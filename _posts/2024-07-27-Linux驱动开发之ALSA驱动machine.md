---
title: Linux驱动开发之ALSA驱动mach
tags: linux 驱动开发 alsa machine
mermaid: true
---

## 概述

**ALSA驱动machine**

## 背景

**基于正点原子的i.MX6ULL开发板，Linux内核版本 `4.1.15`**

## 源文件

> sound/soc/fsl/imx-wm8960.c

```c
static struct snd_soc_dai_link imx_wm8960_dai[] = {
    {
        .name = "HiFi",
        .stream_name = "HiFi",
        .codec_dai_name = "wm8960-hifi",
        .ops = &imx_hifi_ops,
    },
    {
        .name = "HiFi-ASRC-FE",
        .stream_name = "HiFi-ASRC-FE",
        .codec_name = "snd-soc-dummy",
        .codec_dai_name = "snd-soc-dummy-dai",
        .dynamic = 1,
        .ignore_pmdown_time = 1,
        .dpcm_playback = 1,
        .dpcm_capture = 1,
    },
    {
        .name = "HiFi-ASRC-BE",
        .stream_name = "HiFi-ASRC-BE",
        .codec_dai_name = "wm8960-hifi",
        .platform_name = "snd-soc-dummy",
        .no_pcm = 1,
        .ignore_pmdown_time = 1,
        .dpcm_playback = 1,
        .dpcm_capture = 1,
        .ops = &imx_hifi_ops,
        .be_hw_params_fixup = be_hw_params_fixup,
    },
};

static int imx_wm8960_probe(struct platform_device *pdev)
{
    struct device_node *cpu_np, *codec_np = NULL;
    struct platform_device *cpu_pdev;
    struct imx_priv *priv = &card_priv;
    struct i2c_client *codec_dev;
    struct imx_wm8960_data *data;
    struct platform_device *asrc_pdev = NULL;
    struct device_node *asrc_np;
    struct of_phandle_args args;
    u32 width;
    int ret;

    priv->pdev = pdev;
    dev_info(&pdev->dev, "%s\n", __func__);
    dev_dbg(&pdev->dev, "%s\n", __func__);

    cpu_np = of_parse_phandle(pdev->dev.of_node, "cpu-dai", 0);
    if (!cpu_np) {
        dev_err(&pdev->dev, "cpu dai phandle missing or invalid\n");
        ret = -EINVAL;
        goto fail;
    }

    codec_np = of_parse_phandle(pdev->dev.of_node, "audio-codec", 0);
    if (!codec_np) {
        dev_err(&pdev->dev, "phandle missing or invalid\n");
        ret = -EINVAL;
        goto fail;
    }

    cpu_pdev = of_find_device_by_node(cpu_np);
    if (!cpu_pdev) {
        dev_err(&pdev->dev, "failed to find SAI platform device\n");
        ret = -EINVAL;
        goto fail;
    }

    codec_dev = of_find_i2c_device_by_node(codec_np);
    if (!codec_dev || !codec_dev->dev.driver) {
        dev_err(&pdev->dev, "failed to find codec platform device\n");
        ret = -EINVAL;
        goto fail;
    }

    data = devm_kzalloc(&pdev->dev, sizeof(*data), GFP_KERNEL);
    if (!data) {
        ret = -ENOMEM;
        goto fail;
    }

    if (of_property_read_bool(pdev->dev.of_node, "codec-master"))
        data->is_codec_master = true;

    data->codec_clk = devm_clk_get(&codec_dev->dev, "mclk");
    if (IS_ERR(data->codec_clk)) {
        ret = PTR_ERR(data->codec_clk);
        dev_err(&pdev->dev, "failed to get codec clk: %d\n", ret);
        goto fail;
    }

    ret = of_parse_phandle_with_fixed_args(pdev->dev.of_node, "gpr", 3,
                0, &args);
    if (ret) {
        dev_err(&pdev->dev, "failed to get gpr property\n");
        goto fail;
    } else {
        data->gpr = syscon_node_to_regmap(args.np);
        if (IS_ERR(data->gpr)) {
            ret = PTR_ERR(data->gpr);
            dev_err(&pdev->dev, "failed to get gpr regmap\n");
            goto fail;
        }
        regmap_update_bits(data->gpr, args.args[0], args.args[1], args.args[2]);
    }

    of_property_read_u32_array(pdev->dev.of_node, "hp-det", data->hp_det, 2);

    asrc_np = of_parse_phandle(pdev->dev.of_node, "asrc-controller", 0);
    if (asrc_np) {
        asrc_pdev = of_find_device_by_node(asrc_np);
        priv->asrc_pdev = asrc_pdev;
    }

    data->card.dai_link = imx_wm8960_dai;

    imx_wm8960_dai[0].codec_of_node	= codec_np;
    imx_wm8960_dai[0].cpu_dai_name = dev_name(&cpu_pdev->dev);
    imx_wm8960_dai[0].platform_of_node = cpu_np;

    if (!asrc_pdev) {
        data->card.num_links = 1;
    } else {
        imx_wm8960_dai[1].cpu_of_node = asrc_np;
        imx_wm8960_dai[1].platform_of_node = asrc_np;
        imx_wm8960_dai[2].codec_of_node	= codec_np;
        imx_wm8960_dai[2].cpu_dai_name = dev_name(&cpu_pdev->dev);
        data->card.num_links = 3;

        ret = of_property_read_u32(asrc_np, "fsl,asrc-rate",
                &data->asrc_rate);
        if (ret) {
            dev_err(&pdev->dev, "failed to get output rate\n");
            ret = -EINVAL;
            goto fail;
        }

        ret = of_property_read_u32(asrc_np, "fsl,asrc-width", &width);
        if (ret) {
            dev_err(&pdev->dev, "failed to get output rate\n");
            ret = -EINVAL;
            goto fail;
        }

        if (width == 24)
            data->asrc_format = SNDRV_PCM_FORMAT_S24_LE;
        else
            data->asrc_format = SNDRV_PCM_FORMAT_S16_LE;
    }

    data->card.dev = &pdev->dev;
    data->card.owner = THIS_MODULE;
    ret = snd_soc_of_parse_card_name(&data->card, "model");
    if (ret)
        goto fail;
    data->card.dapm_widgets = imx_wm8960_dapm_widgets;
    data->card.num_dapm_widgets = ARRAY_SIZE(imx_wm8960_dapm_widgets);

    ret = snd_soc_of_parse_audio_routing(&data->card, "audio-routing");
    if (ret)
        goto fail;

    data->card.late_probe = imx_wm8960_late_probe;

    platform_set_drvdata(pdev, &data->card);
    snd_soc_card_set_drvdata(&data->card, data);
    ret = devm_snd_soc_register_card(&pdev->dev, &data->card);
    if (ret) {
        dev_err(&pdev->dev, "snd_soc_register_card failed (%d)\n", ret);
        goto fail;
    }

    priv->snd_card = data->card.snd_card;

    imx_hp_jack_gpio.gpio = of_get_named_gpio_flags(pdev->dev.of_node,
            "hp-det-gpios", 0, &priv->hp_active_low);

    imx_mic_jack_gpio.gpio = of_get_named_gpio_flags(pdev->dev.of_node,
            "mic-det-gpios", 0, &priv->mic_active_low);

    if (gpio_is_valid(imx_hp_jack_gpio.gpio) &&
        gpio_is_valid(imx_mic_jack_gpio.gpio) &&
        imx_hp_jack_gpio.gpio == imx_mic_jack_gpio.gpio)
        priv->is_headset_jack = true;

    if (gpio_is_valid(imx_hp_jack_gpio.gpio)) {
        priv->headphone_kctl = snd_kctl_jack_new("Headphone", 0, NULL);
        ret = snd_ctl_add(priv->snd_card, priv->headphone_kctl);
        if (ret)
            dev_warn(&pdev->dev, "failed to create headphone jack kctl\n");

        if (priv->is_headset_jack) {
            imx_hp_jack_pin.mask |= SND_JACK_MICROPHONE;
            imx_hp_jack_gpio.report |= SND_JACK_MICROPHONE;
        }
        imx_hp_jack_gpio.jack_status_check = hp_jack_status_check;
        imx_hp_jack_gpio.data = &imx_hp_jack;
        ret = imx_wm8960_jack_init(&data->card, &imx_hp_jack,
                       &imx_hp_jack_pin, &imx_hp_jack_gpio);
        if (ret) {
            dev_warn(&pdev->dev, "hp jack init failed (%d)\n", ret);
            goto out;
        }

        ret = driver_create_file(pdev->dev.driver, &driver_attr_headphone);
        if (ret)
            dev_warn(&pdev->dev, "create hp attr failed (%d)\n", ret);
    }

    if (gpio_is_valid(imx_mic_jack_gpio.gpio)) {
        if (!priv->is_headset_jack) {
            imx_mic_jack_gpio.jack_status_check = mic_jack_status_check;
            imx_mic_jack_gpio.data = &imx_mic_jack;
            ret = imx_wm8960_jack_init(&data->card, &imx_mic_jack,
                    &imx_mic_jack_pins, &imx_mic_jack_gpio);
            if (ret) {
                dev_warn(&pdev->dev, "mic jack init failed (%d)\n", ret);
                goto out;
            }
        }
        ret = driver_create_file(pdev->dev.driver, &driver_attr_micphone);
        if (ret)
            dev_warn(&pdev->dev, "create mic attr failed (%d)\n", ret);
    }

out:
    ret = 0;
fail:
    if (cpu_np)
        of_node_put(cpu_np);
    if (codec_np)
        of_node_put(codec_np);

    return ret;
}
```

```c
devm_snd_soc_register_card(&pdev->dev, &data->card);
--> devres_alloc(devm_card_release, sizeof(*ptr), GFP_KERNEL);
--> snd_soc_register_card(card);
----> snd_soc_init_multicodec(card, link);
----> snd_soc_initialize_card_lists(card);
----> snd_soc_instantiate_card(card);
------> soc_bind_dai_link(card, i);
--------> rtd->cpu_dai = snd_soc_find_dai(&cpu_dai_component);
----------> list_for_each_entry(component, &component_list, list) {/*匹配*/}
--------> codec_dais[i] = snd_soc_find_dai(&codecs[i]);
----------> list_for_each_entry(component, &component_list, list) {/*匹配*/}
------> snd_card_new(card->dev, SNDRV_DEFAULT_IDX1, SNDRV_DEFAULT_STR1, card->owner, 0, &card->snd_card);
------> soc_init_card_debugfs(card);
------> snd_soc_dapm_debugfs_init(&card->dapm, card->debugfs_card_root);
------> snd_soc_dapm_new_controls(&card->dapm, card->dapm_widgets, card->num_dapm_widgets);
------> snd_soc_dapm_new_controls(&card->dapm, card->of_dapm_widgets, card->num_of_dapm_widgets);
------> card->probe(card);
------> soc_probe_link_components(card, i, order);
------> soc_probe_link_dais(card, i, order);
------> soc_probe_aux_dev(card, i);
------> snd_soc_dapm_link_dai_widgets(card);
------> snd_soc_dapm_connect_dai_link_widgets(card);
------> snd_soc_add_card_controls(card, card->controls, card->num_controls);
------> snd_soc_dapm_add_routes(&card->dapm, card->dapm_routes,card->num_dapm_routes);
------> snd_soc_dapm_add_routes(&card->dapm, card->of_dapm_routes,card->num_of_dapm_routes);
------> card->late_probe(card);
------> snd_soc_dapm_new_widgets(card);
------> snd_card_register(card->snd_card);
------> snd_soc_dapm_sync(&card->dapm);
--> devres_add;
```
