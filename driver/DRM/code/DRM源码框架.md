# 4G/5G平台区别
- 4G的display相关code主要位于：drivers/misc/mediatek/video/mtxxxx/dispsys
- 4G平台会在drivers/misc/mediatek/video/建立一堆mtxxxx来区分不同的chip
- 而且4G平台采用分chip管理，代码重复，像Talbot，zion等4G平台都没有使用DRM，而是MTK自己的一套框架
- 4G平台由于会给每个chip创建mtxxxx/目录，所以代码比较集中，比如cmdq、dispsys、m4u、videox等，这样更容易去理解各模块功能
- 5G平台做了共code处理，相关display code放在drivers/gpu/drm/mediatek/目录
- 如果5G平台确实有需要做chip区分的，那也是在代码中插入条件编译
- 新的5G平台编译结果中已经没有了drivers/misc/mediatek/video这个目录
- 而5G平台厂商主要关注的目录就是drivers/gpu/drm/panel/和drivers/gpu/drm/mediatek/这两个目录



# 模块划分
## MMSYS
- Coordinator
- MMSYS TOP
## MDP
- MDP_RDMA
- MDP_WROT
- MDP_TDSHP
- MDP_RSZ
## DISP
- DISP_OVL
- DISP_RDMA
- DISP_WDMA
- DISP_DBI
- DISP_DSI
## PQ
- DISP_AAL
- DISP_COLOR
- DISP_CCORR
- DISP_GAMMA
- DISP_DITHER
## GCE
- GCE
- mutext
- DISP_PWM
## SMI
- SMI_TOP (architecture)
- SMI_TOP (RTL)
- SMI_COMMON
- SMI_LARB
- IOMMU




# drm 代码划分
## drm core
路径：drivers/gpu/drm/
- drm_framebuffer.c
- drm_crtc.c
- drm_crtc_helper.c
- drm_fb_helper.c
- drm_file.c
- drm_gem.c
- drm_kms_helper.c
- drm_panel.c
- drm_plane.c

## MTK drm core
路径：drivers/gpu/drm/mediatek/
- mtk_drm_drv.c

## MTK drm framework related
- 这里的framework是DRM的框架，不是Android的framework
路径：drivers/gpu/drm/mediatek/
- mtk_drm_crtc.c
- mtk_drm_plane.c
- mtk_drm_fb.c
- mtk_drm_gem.c

mtk_drm_drv.c -> mtk_drm_kms_init() -> mtk_drm_crtc.c -> mtk_drm_plane.c
mtk_drm_drv.c -> mtk_drm_kms_init() -> mtk_drm_fb.c

## MTK hardware related
路径：drivers/gpu/drm/mediatek/
- mtk_drm_ddp.c
- mtk_drm_ddp_comp.c
- mtk_disp_ovl.c
- mtk_disp_rdma.c
- mtk_disp_color.c
- mtk_disp_aal.c

## cmdq
路径：
drivers/misc/mediatek/cmdq
drivers/gpu/drm/mediatek/
- mtk_cmdq_dummy.c

## Connector/encoder
路径：drivers/gpu/drm/mediatek/
- mtk_dsi.c/mtk_lvds.c, …,etc
## Debug
路径：drivers/gpu/drm/mediatek/
- mtk_drm_debugfs.c
## Panel
路径：drivers/gpu/drm/panel/
- panel-simple.c
## Other


# MTK drm core分析
## mtk_drm_drv.c介绍
- 模块加载时统一注册多个driver：
    - mtk_drm_init() ->platform_driver_register(mtk_drm_drivers[i]);
    - mtk_drm_exit() ->platform_driver_unregister(mtk_drm_drivers[i]);
- mtk_drm_drivers[ ]：(每个Display模块都有独立的driver,没写HDMI)
    - mtk_drm_platform_driver,
    - mtk_ddp_driver,
    - mtk_disp_color_driver,
    - mtk_disp_ccorr_driver,
    - mtk_disp_gamma_driver,
    - mtk_disp_aal_driver,
    - mtk_dmdp_aal_driver,
    - mtk_disp_postmask_driver,
    - mtk_disp_dither_driver,
    - mtk_disp_ovl_driver,
    - mtk_disp_rdma_driver,
    - mtk_disp_wdma_driver,
    - mtk_disp_rsz_driver,
    - mtk_dsi_driver,
    - mtk_mipi_tx_driver,
    - mtk_disp_dsc_driver,
    - mtk_disp_merge_driver
- 不管Display模块有多少个驱动，但对外表现的设备文件或sys文件是有限的，所以得找到负责对外的设备驱动，这些驱动是关联其它驱动的基础
- DRM driver入口：drivers/gpu/drm/drm_drv.c

## 传统的/dev/fb0实现
- drivers/gpu/drm/mediatek/mtk_drm_fbdev.c->mtk_fbdev_probe()->
- drivers/gpu/drm/drm_fb_helper.c->drm_fb_helper_alloc_fbi()
- 但建立关联的fb_info好像与fbmem.c没什么关系
- 所以不知道/dev/fb0这种方式有没有实现，似乎独立于fbmem.c重新实现了，具体流程如下：
    - mtk_drm_init()->platform_driver_register()->mtk_drm_platform_driver->mtk_drm_probe()->component_master_add_with_match()->->component_master_ops->mtk_drm_bind()->申请并初始化drm_device然后一层层传入->mtk_drm_kms_init()->mtk_fbdev_init()->drm_fb_helper_prepare(&mtk_drm_fb_helper_funcs)
    - mtk_drm_fb_helper_funcs->mtk_fbdev_probe()->drm_fb_helper_alloc_fbi()->申请fb_info并挂在了drm_fb_helper的fbdev成员下面->drm_fb_helper来至于传入的drm_device 

component.c机制
component_master_add_with_match()就是决定绑定的模块加载先后顺序，是内核提供的与dts结合的一个模块






driver实例概述
drm驱动程序的设备实例由&struct drm_device表示。这是通过drm_dev_alloc()分配的，通常是由驱动程序实现的总线特定的-&gt;probe()回调。然后驱动程序需要初始化drm设备的所有各个子系统，如内存管理、vblank处理、modeset支持和初始输出配置，显然还要初始化所有相应的硬件位。其中一个重要的部分是调用drm_dev_set_unique()来设置这个设备实例的用户空间可见的惟一名称。最后，当一切就绪并运行并为用户空间做好准备时，可以使用drm_dev_register()发布设备实例。
还有使用总线特定的帮助程序和&drm_driver初始化设备实例的已弃用支持。负载调。但是由于向后兼容的需要，设备实例必须过早发布，这需要不太漂亮的全局锁来确保安全，因此只支持尚未转换到新方案的现有驱动程序。
当清理一个设备实例时，所有的事情都需要反向执行:
首先使用drm_dev_unregister()取消发布设备实例。然后清理在设备初始化时分配的任何其他资源，并使用drm_dev_unref()删除驱动程序对&drm_device的引用。注意，&drm_device实例的生存期规则仍然有很多历史包袱。因此，请谨慎使用drm_dev_ref()和drm_dev_unref()提供的引用计数。建议驱动程序将drm_device嵌入到它们自己的设备结构中，这是通过drm_dev_init()支持的。








# ref
- Display study guide：https://wiki.mediatek.inc/display/~MTK33237/Copy+of+Display+Study+guide
