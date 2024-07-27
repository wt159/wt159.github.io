---
title: Linux驱动开发之ALSA驱动platform
tags: linux 驱动开发 alsa platform
mermaid: true
---

## 概述

**ALSA驱动platform**

## 背景

**基于正点原子的i.MX6ULL开发板，Linux内核版本 `4.1.15`**

## 源文件

> sound/soc/fsl/fsl_sai.c

```c
static struct snd_soc_dai_driver fsl_sai_dai = {
    .probe = fsl_sai_dai_probe,
    .playback = {
        .stream_name = "CPU-Playback",
        .channels_min = 1,
        .channels_max = 2,
        .rate_min = 8000,
        .rate_max = 192000,
        .rates = SNDRV_PCM_RATE_KNOT,
        .formats = FSL_SAI_FORMATS,
    },
    .capture = {
        .stream_name = "CPU-Capture",
        .channels_min = 1,
        .channels_max = 2,
        .rate_min = 8000,
        .rate_max = 192000,
        .rates = SNDRV_PCM_RATE_KNOT,
        .formats = FSL_SAI_FORMATS,
    },
    .ops = &fsl_sai_pcm_dai_ops,
};
static const struct snd_soc_component_driver fsl_component = {
    .name           = "fsl-sai",
};
static int fsl_sai_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    struct fsl_sai *sai;
    struct resource *res;
    void __iomem *base;
    char tmp[8];
    int irq, ret, i;
    u32 buffer_size;

    dev_info(&pdev->dev, "%s: entor\n", __func__);

    sai = devm_kzalloc(&pdev->dev, sizeof(*sai), GFP_KERNEL);
    if (!sai)
        return -ENOMEM;

    sai->pdev = pdev;

    if (of_device_is_compatible(pdev->dev.of_node, "fsl,imx6sx-sai"))
        sai->sai_on_imx = true;

    sai->is_lsb_first = of_property_read_bool(np, "lsb-first");

    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(base))
        return PTR_ERR(base);

    sai->regmap = devm_regmap_init_mmio_clk(&pdev->dev,
            "bus", base, &fsl_sai_regmap_config);

    /* Compatible with old DTB cases */
    if (IS_ERR(sai->regmap))
        sai->regmap = devm_regmap_init_mmio_clk(&pdev->dev,
                "sai", base, &fsl_sai_regmap_config);
    if (IS_ERR(sai->regmap)) {
        dev_err(&pdev->dev, "regmap init failed\n");
        return PTR_ERR(sai->regmap);
    }

    /* No error out for old DTB cases but only mark the clock NULL */
    sai->bus_clk = devm_clk_get(&pdev->dev, "bus");
    if (IS_ERR(sai->bus_clk)) {
        dev_err(&pdev->dev, "failed to get bus clock: %ld\n",
                PTR_ERR(sai->bus_clk));
        sai->bus_clk = NULL;
    }

    for (i = 0; i < FSL_SAI_MCLK_MAX; i++) {
        sprintf(tmp, "mclk%d", i);
        sai->mclk_clk[i] = devm_clk_get(&pdev->dev, tmp);
        if (IS_ERR(sai->mclk_clk[i])) {
            dev_err(&pdev->dev, "failed to get mclk%d clock: %ld\n",
                    i + 1, PTR_ERR(sai->mclk_clk[i]));
            sai->mclk_clk[i] = NULL;
        }
    }

    sai->slots = 2;
    sai->slot_width = 32;

    irq = platform_get_irq(pdev, 0);
    if (irq < 0) {
        dev_err(&pdev->dev, "no irq for node %s\n", pdev->name);
        return irq;
    }

    ret = devm_request_irq(&pdev->dev, irq, fsl_sai_isr, 0, np->name, sai);
    if (ret) {
        dev_err(&pdev->dev, "failed to claim irq %u\n", irq);
        return ret;
    }

    /* Sync Tx with Rx as default by following old DT binding */
    sai->synchronous[RX] = true;
    sai->synchronous[TX] = false;
    fsl_sai_dai.symmetric_rates = 1;
    fsl_sai_dai.symmetric_channels = 1;
    fsl_sai_dai.symmetric_samplebits = 1;

    if (of_find_property(np, "fsl,sai-synchronous-rx", NULL) &&
        of_find_property(np, "fsl,sai-asynchronous", NULL)) {
        /* error out if both synchronous and asynchronous are present */
        dev_err(&pdev->dev, "invalid binding for synchronous mode\n");
        return -EINVAL;
    }

    if (of_find_property(np, "fsl,sai-synchronous-rx", NULL)) {
        /* Sync Rx with Tx */
        sai->synchronous[RX] = false;
        sai->synchronous[TX] = true;
    } else if (of_find_property(np, "fsl,sai-asynchronous", NULL)) {
        /* Discard all settings for asynchronous mode */
        sai->synchronous[RX] = false;
        sai->synchronous[TX] = false;
        fsl_sai_dai.symmetric_rates = 0;
        fsl_sai_dai.symmetric_channels = 0;
        fsl_sai_dai.symmetric_samplebits = 0;
    }

    sai->dma_params_rx.addr = res->start + FSL_SAI_RDR;
    sai->dma_params_tx.addr = res->start + FSL_SAI_TDR;
    sai->dma_params_rx.maxburst = FSL_SAI_MAXBURST_RX;
    sai->dma_params_tx.maxburst = FSL_SAI_MAXBURST_TX;

    platform_set_drvdata(pdev, sai);

    pm_runtime_enable(&pdev->dev);

    ret = devm_snd_soc_register_component(&pdev->dev, &fsl_component,
            &fsl_sai_dai, 1);
    if (ret)
        return ret;

    if (of_property_read_u32(np, "fsl,dma-buffer-size", &buffer_size))
        buffer_size = IMX_SAI_DMABUF_SIZE;

    if (sai->sai_on_imx)
        return imx_pcm_dma_init(pdev, buffer_size);
    else
        return devm_snd_dmaengine_pcm_register(&pdev->dev, NULL,
                SND_DMAENGINE_PCM_FLAG_NO_RESIDUE);
}
static struct platform_driver fsl_sai_driver = {
    .probe = fsl_sai_probe,
    .driver = {
        .name = "fsl-sai",
        .pm = &fsl_sai_pm_ops,
        .of_match_table = fsl_sai_ids,
    },
};
module_platform_driver(fsl_sai_driver);
```

```c
devm_snd_soc_register_component(&pdev->dev, &fsl_component, &fsl_sai_dai, 1);
--> devres_alloc(devm_component_release, sizeof(*ptr), GFP_KERNEL);
--> snd_soc_register_component(dev, cmpnt_drv, dai_drv, num_dai);
----> cmpnt = kzalloc(sizeof(*cmpnt), GFP_KERNEL);
----> snd_soc_component_initialize(cmpnt, cmpnt_drv, dev);
----> snd_soc_register_dais(cmpnt, dai_drv, num_dai, true);
----> snd_soc_component_add(cmpnt);
------> snd_soc_component_add_unlocked(component);
--------> list_add(&component->list, &component_list);
--> devres_add(dev, ptr);
```
