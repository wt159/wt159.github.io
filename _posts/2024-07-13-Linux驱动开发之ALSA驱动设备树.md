---
title: Linux驱动开发之ALSA驱动设备树
tags: linux 驱动开发 alsa 设备树
mermaid: true
---

## 概述

ALSA（Advanced Linux Sound Architecture）是Linux内核中关于音频驱动的框架，它定义了音频驱动的框架，使得音频驱动可以方便地被用户空间调用。

ALSA驱动起始页[https://kernel.org/doc/html/latest/sound/index.html](https://kernel.org/doc/html/latest/sound/index.html)

asla驱动代码结构[https://kernel.org/doc/html/latest/sound/kernel-api/writing-an-alsa-driver.html](https://kernel.org/doc/html/latest/sound/kernel-api/writing-an-alsa-driver.html)

本文主要介绍一下ALSA驱动开发的相关设备树及源文件。

## 背景

**基于正点原子的i.MX6ULL开发板，Linux内核版本 `4.1.15`**

## 设备树 `imx6ull-alientek-emmc.dts`

```dts
sound {
    compatible = "fsl,imx6ul-evk-wm8960",
            "fsl,imx-audio-wm8960";
    model = "wm8960-audio";
    cpu-dai = <&sai2>;
    audio-codec = <&codec>;
    asrc-controller = <&asrc>;
    codec-master;
    gpr = <&gpr 4 0x100000 0x100000>;
    /*
                * hp-det = <hp-det-pin hp-det-polarity>;
        * hp-det-pin: JD1 JD2  or JD3
        * hp-det-polarity = 0: hp detect high for headphone
        * hp-det-polarity = 1: hp detect high for speaker
        */
    hp-det = <3 0>;
    hp-det-gpios = <&gpio5 4 0>;
    mic-det-gpios = <&gpio5 4 0>;
    audio-routing =
        "Headphone Jack", "HP_L",
        "Headphone Jack", "HP_R",
        "Ext Spk", "SPK_LP",
        "Ext Spk", "SPK_LN",
        "Ext Spk", "SPK_RP",
        "Ext Spk", "SPK_RN",
        "LINPUT2", "Mic Jack",
        "LINPUT3", "Mic Jack",
        "RINPUT1", "Main MIC",
        "RINPUT2", "Main MIC",
        "Mic Jack", "MICB",
        "Main MIC", "MICB",
        "CPU-Playback", "ASRC-Playback",
        "Playback", "CPU-Playback",
        "ASRC-Capture", "CPU-Capture",
        "CPU-Capture", "Capture";
};

&i2c1 {
    clock-frequency = <100000>;
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_i2c1>;
    status = "okay";

    codec: wm8960@1a {
        compatible = "wlf,wm8960";
        reg = <0x1a>;
        clocks = <&clks IMX6UL_CLK_SAI2>;
        clock-names = "mclk";
        wlf,shared-lrclk;
    };
};

&sai2 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_sai2
             &pinctrl_sai2_hp_det_b>;

    assigned-clocks = <&clks IMX6UL_CLK_SAI2_SEL>,
              <&clks IMX6UL_CLK_SAI2>;
    assigned-clock-parents = <&clks IMX6UL_CLK_PLL4_AUDIO_DIV>;
    assigned-clock-rates = <0>, <12288000>;

    status = "okay";
};

```

## 次级设备树 `imx6ull.dtsi`

```dts
sai2: sai@0202c000 {
    compatible = "fsl,imx6ul-sai",
                "fsl,imx6sx-sai";
    reg = <0x0202c000 0x4000>;
    interrupts = <GIC_SPI 98 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clks IMX6UL_CLK_SAI2_IPG>,
            <&clks IMX6UL_CLK_DUMMY>,
            <&clks IMX6UL_CLK_SAI2>,
            <&clks 0>, <&clks 0>;
    clock-names = "bus", "mclk0", "mclk1", "mclk2", "mclk3";
    dma-names = "rx", "tx";
    dmas = <&sdma 37 24 0>, <&sdma 38 24 0>;
    status = "okay";
};

asrc: asrc@02034000 {
    compatible = "fsl,imx53-asrc";
    reg = <0x02034000 0x4000>;
    interrupts = <GIC_SPI 50 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clks IMX6UL_CLK_ASRC_IPG>,
        <&clks IMX6UL_CLK_ASRC_MEM>, <&clks 0>,
        <&clks 0>, <&clks 0>, <&clks 0>, <&clks 0>,
        <&clks 0>, <&clks 0>, <&clks 0>, <&clks 0>,
        <&clks 0>, <&clks 0>, <&clks 0>, <&clks 0>,
        <&clks IMX6UL_CLK_SPDIF>, <&clks 0>, <&clks 0>,
        <&clks IMX6UL_CLK_SPBA>;
    clock-names = "mem", "ipg", "asrck_0",
        "asrck_1", "asrck_2", "asrck_3", "asrck_4",
        "asrck_5", "asrck_6", "asrck_7", "asrck_8",
        "asrck_9", "asrck_a", "asrck_b", "asrck_c",
        "asrck_d", "asrck_e", "asrck_f", "dma";
    dmas = <&sdma 17 23 1>, <&sdma 18 23 1>, <&sdma 19 23 1>,
        <&sdma 20 23 1>, <&sdma 21 23 1>, <&sdma 22 23 1>;
    dma-names = "rxa", "rxb", "rxc",
            "txa", "txb", "txc";
    fsl,asrc-rate  = <48000>;
    fsl,asrc-width = <16>;
    status = "okay";
};

sdma: sdma@020ec000 {
    compatible = "fsl,imx6ul-sdma", "fsl,imx35-sdma";
    reg = <0x020ec000 0x4000>;
    interrupts = <GIC_SPI 2 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clks IMX6UL_CLK_SDMA>,
            <&clks IMX6UL_CLK_SDMA>;
    clock-names = "ipg", "ahb";
    #dma-cells = <3>;
    iram = <&ocram>;
    fsl,sdma-ram-script-name = "imx/sdma/sdma-imx6q.bin";
};
```

## 源文件

```C
//sound/soc/fsl/imx-wm8960.c
static const struct of_device_id imx_wm8960_dt_ids[] = {
    { .compatible = "fsl,imx-audio-wm8960", },
    { /* sentinel */ }
};
static struct platform_driver imx_wm8960_driver = {
    .driver = {
        .name = "imx-wm8960",
        .pm = &snd_soc_pm_ops,
        .of_match_table = imx_wm8960_dt_ids,
    },
    .probe = imx_wm8960_probe,
    .remove = imx_wm8960_remove,
};
module_platform_driver(imx_wm8960_driver);
```

```C
//sound/soc/fsl/fsl_sai.c
static const struct of_device_id fsl_sai_ids[] = {
    { .compatible = "fsl,vf610-sai", },
    { .compatible = "fsl,imx6sx-sai", },
    { /* sentinel */ }
};
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

```C
//sound/soc/fsl/fsl_asrc.c
static const struct of_device_id fsl_asrc_ids[] = {
    { .compatible = "fsl,imx35-asrc", },
    { .compatible = "fsl,imx53-asrc", },
    {}
};
static struct platform_driver fsl_asrc_driver = {
    .probe = fsl_asrc_probe,
    .remove = fsl_asrc_m2m_remove,
    .driver = {
        .name = "fsl-asrc",
        .of_match_table = fsl_asrc_ids,
        .pm = &fsl_asrc_pm,
    },
};
module_platform_driver(fsl_asrc_driver);
```

```C
//sound/soc/codecs/wm8960.c
static const struct of_device_id wm8960_of_match[] = {
       { .compatible = "wlf,wm8960", },
       { }
};
static struct i2c_driver wm8960_i2c_driver = {
    .driver = {
        .name = "wm8960",
        .owner = THIS_MODULE,
        .of_match_table = wm8960_of_match,
    },
    .probe =    wm8960_i2c_probe,
    .remove =   wm8960_i2c_remove,
    .id_table = wm8960_i2c_id,
};
module_i2c_driver(wm8960_i2c_driver);
```
