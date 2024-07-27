---
title: Linux驱动开发之ALSA驱动codec
tags: linux 驱动开发 alsa codec
mermaid: true
---

## 概述

**ALSA驱动codec**

## 背景

**基于正点原子的i.MX6ULL开发板，Linux内核版本 `4.1.15`**

## 源文件

> sound/soc/codecs/wm8960.c

```c
static struct snd_soc_dai_driver wm8960_dai = {
    .name = "wm8960-hifi",
    .playback = {
        .stream_name = "Playback",
        .channels_min = 1,
        .channels_max = 2,
        .rates = WM8960_RATES,
        .formats = WM8960_FORMATS,},
    .capture = {
        .stream_name = "Capture",
        .channels_min = 1,
        .channels_max = 2,
        .rates = WM8960_RATES,
        .formats = WM8960_FORMATS,},
    .ops = &wm8960_dai_ops,
    .symmetric_rates = 1,
};

static struct snd_soc_codec_driver soc_codec_dev_wm8960 = {
    .probe = wm8960_probe,
    .set_bias_level = wm8960_set_bias_level,
    .suspend_bias_off = true,
};

static int wm8960_i2c_probe(struct i2c_client *i2c,
                const struct i2c_device_id *id)
{
    struct wm8960_data *pdata = dev_get_platdata(&i2c->dev);
    struct wm8960_priv *wm8960;
    int ret;
    int repeat_reset = 10;

    wm8960 = devm_kzalloc(&i2c->dev, sizeof(struct wm8960_priv),
                  GFP_KERNEL);
    if (wm8960 == NULL)
        return -ENOMEM;

    wm8960->mclk = devm_clk_get(&i2c->dev, "mclk");
    if (IS_ERR(wm8960->mclk)) {
        if (PTR_ERR(wm8960->mclk) == -EPROBE_DEFER)
            return -EPROBE_DEFER;
    }

    wm8960->regmap = devm_regmap_init_i2c(i2c, &wm8960_regmap);
    if (IS_ERR(wm8960->regmap))
        return PTR_ERR(wm8960->regmap);

    if (pdata)
        memcpy(&wm8960->pdata, pdata, sizeof(struct wm8960_data));
    else if (i2c->dev.of_node)
        wm8960_set_pdata_from_of(i2c, &wm8960->pdata);

    do {
        ret = wm8960_reset(wm8960->regmap);
        repeat_reset--;
    } while (repeat_reset > 0 && ret != 0);

    if (ret != 0) {
        dev_err(&i2c->dev, "Failed to issue reset\n");
        return ret;
    }

    if (wm8960->pdata.shared_lrclk) {
        ret = regmap_update_bits(wm8960->regmap, WM8960_ADDCTL2,
                     0x4, 0x4);
        if (ret != 0) {
            dev_err(&i2c->dev, "Failed to enable LRCM: %d\n",
                ret);
            return ret;
        }
    }

    /* Latch the update bits */
    regmap_update_bits(wm8960->regmap, WM8960_LINVOL, 0x100, 0x100);
    regmap_update_bits(wm8960->regmap, WM8960_RINVOL, 0x100, 0x100);
    regmap_update_bits(wm8960->regmap, WM8960_LADC, 0x100, 0x100);
    regmap_update_bits(wm8960->regmap, WM8960_RADC, 0x100, 0x100);
    regmap_update_bits(wm8960->regmap, WM8960_LDAC, 0x100, 0x100);
    regmap_update_bits(wm8960->regmap, WM8960_RDAC, 0x100, 0x100);
    regmap_update_bits(wm8960->regmap, WM8960_LOUT1, 0x100, 0x100);
    regmap_update_bits(wm8960->regmap, WM8960_ROUT1, 0x100, 0x100);
    regmap_update_bits(wm8960->regmap, WM8960_LOUT2, 0x100, 0x100);
    regmap_update_bits(wm8960->regmap, WM8960_ROUT2, 0x100, 0x100);

    i2c_set_clientdata(i2c, wm8960);

    ret = snd_soc_register_codec(&i2c->dev,
            &soc_codec_dev_wm8960, &wm8960_dai, 1);

    return ret;
}
```

```c
snd_soc_register_codec(&i2c->dev, &soc_codec_dev_wm8960, &wm8960_dai, 1);
--> kzalloc(sizeof(struct snd_soc_codec), GFP_KERNEL);
--> snd_soc_component_initialize(&codec->component, &codec_drv->component_driver, dev);
--> snd_soc_register_dais(&codec->component, dai_drv, num_dai, false);
--> snd_soc_component_add_unlocked(&codec->component);
----> list_add(&component->list, &component_list);
--> list_add(&codec->list, &codec_list);;
```
