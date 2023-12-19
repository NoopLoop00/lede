# ZTE E8820v2固件编译

首先感谢P3TERX、coolsnowwolf和siwind三位大佬的Aciton库、源码和dts

Action库 https://github.com/P3TERX/Actions-OpenWrt

源码 https://github.com/coolsnowwolf/lede

dts https://github.com/siwind/openwrt/tree/master/target/linux/ramips/dts

## 进行Aciton准备工作


1.因为源码中没有e8820v2的dts文件和机型定义，所以首先要将Action和源码folk，然后下载siwind大佬制作的dts文件，链接上面已经给到，将dts稍加编辑后加入到/target/linux/ramips/dts内，也可直接下载本仓库内的dts。

```bash
// SPDX-License-Identifier: GPL-2.0-or-later OR MIT

#include "mt7621.dtsi"

#include <dt-bindings/gpio/gpio.h>
#include <dt-bindings/input/input.h>

/ {
	compatible = "zte,e8820v2", "mediatek,mt7621-soc";
	model = "ZTE E8820V2";

	aliases {
		led-boot = &led_sys;
		led-failsafe = &led_sys;
		led-running = &led_sys;
		led-upgrade = &led_sys;
		label-mac-device = &gmac0;
	};

	chosen {
		bootargs = "console=ttyS0,115200";
	};

	leds {
		compatible = "gpio-leds";

		led_sys: sys {
			label = "white:sys";
			gpios = <&gpio 29 GPIO_ACTIVE_LOW>;
		};

		power {
			label = "white:power";
			gpios = <&gpio 31 GPIO_ACTIVE_LOW>;
		};
	};

	keys {
		compatible = "gpio-keys";

		reset {
			label = "reset";
			gpios = <&gpio 18 GPIO_ACTIVE_LOW>;
			linux,code = <KEY_RESTART>;
		};

		wps {
			label = "wps";
			gpios = <&gpio 24 GPIO_ACTIVE_LOW>;
			linux,code = <KEY_WPS_BUTTON>;
		};
	};
};

&spi0 {
	status = "okay";

	flash@0 {
		compatible = "jedec,spi-nor";
		reg = <0>;
		spi-max-frequency = <25000000>;
		broken-flash-reset;
		m25p,fast-read;

		partitions {
			compatible = "fixed-partitions";
			#address-cells = <1>;
			#size-cells = <1>;

			partition@0 {
				label = "u-boot";
				reg = <0x0 0x30000>;
				read-only;
			};

			partition@30000 {
				label = "u-boot-env";
				reg = <0x30000 0x10000>;
				read-only;
			};

			factory: partition@40000 {
				label = "factory";
				reg = <0x40000 0x10000>;
				read-only;
			};

			partition@50000 {
				compatible = "denx,uimage";
				label = "firmware";
				reg = <0x50000 0x1fb0000>;
			};
		};
	};
};

&pcie {
	status = "okay";
	reset-gpios = <&gpio 19 GPIO_ACTIVE_LOW>,
				  <&gpio 4 GPIO_ACTIVE_LOW>;
};

&pcie0 {
	wifi@0,0 {
		reg = <0x0000 0 0 0 0>;
		mediatek,mtd-eeprom = <&factory 0x0000>;
		mtd-mac-address = <&factory 0xe000>;

		led {
			led-active-low;
		};
	};

};

&pcie1 {
	wifi@0,0 {
		reg = <0x0000 0 0 0 0>;
		mediatek,mtd-eeprom = <&factory 0x8000>;
		mtd-mac-address = <&factory 0xe006>;
		ieee80211-freq-limit = <5000000 6000000>;

		led {
			led-sources = <2>;
			led-active-low;
		};
	};

};

&gmac0 {
	mtd-mac-address = <&factory 0xe000>;
};

&gsw {
	mediatek,portmap = "llllw";
	status = "okay";
};

&hnat {
	mtketh-wan = "eth0.2";
	mtketh-ppd = "eth0.1";
	mtketh-lan = "eth0";
	mtketh-max-gmac = <1>;
	/delete-property/ mtkdsa-wan-port;
};

&switch0 {
	status = "disabled";
};

&state_default {
	gpio {
		groups = "i2c", "uart2", "uart3", "wdt";
		function = "gpio";
	};
};
```



编辑/target/linux/ramips/image下的mt7621.mk，在最后添加e8820v2的机型定义，如下:

   ```bash
   define Device/zte_e8820v2
     $(Device/dsa-migration)
     $(Device/uimage-lzma-loader)
     IMAGE_SIZE := 16064k
     DEVICE_VENDOR := ZTE
     DEVICE_MODEL := E8820V2
     DEVICE_COMPAT_VERSION := 2.0
     DEVICE_PACKAGES := kmod-mt7603e kmod-mt76x2e kmod-usb2 \
   	  kmod-usb-ledtrig-usbport luci-app-mtwifi -wpad-openssl
   endef
   TARGET_DEVICES += zte_e8820v2
   ```

——————————————————————————————————————————————————————————————
PS:如果是硬改的32m的rom，需要修改dts和机型代码，dts文件中这处字段需要修改：

   ```bash
   partition@50000 {
	   			compatible = "denx,uimage";
		   		label = "firmware";
		   		reg = <0x50000 0x1fb0000>;
		   	};

   # 将0xfb0000修改为0x1fb0000
   ```

机型代码修改如下：

   ```bash
   define Device/zte_e8820v2
     $(Device/dsa-migration)
     $(Device/uimage-lzma-loader)
     IMAGE_SIZE := 32448k
     DEVICE_VENDOR := ZTE
     DEVICE_MODEL := E8820V2
     DEVICE_COMPAT_VERSION := 2.0
     DEVICE_PACKAGES := kmod-mt7603e kmod-mt76x2e kmod-usb2 \
   	  kmod-usb-ledtrig-usbport luci-app-mtwifi -wpad-openssl
   endef
   TARGET_DEVICES += zte_e8820v2

   # 将IMAGE_SIZE的值修改为32448k
   ```
——————————————————————————————————————————————————————————————

### 编辑Action库

修改/.github/workflows/build-openwrt.yml文件:

   ```bash
  env:
  REPO_URL: https://github.com/yourname/lede.git
  REPO_BRANCH: master
  FEEDS_CONF: feeds.conf.default
  CONFIG_FILE: .config
  DIY_P1_SH: diy-part1.sh
  DIY_P2_SH: diy-part2.sh
  UPLOAD_BIN_DIR: false
  UPLOAD_FIRMWARE: true
  UPLOAD_COWTRANSFER: false
  UPLOAD_WETRANSFER: false
  UPLOAD_RELEASE: false
  TZ: Asia/Shanghai

  # REPO_URL的值修改为你的源码仓库地址
  # CONFIG_FILE的值修改为你的配置文件命
   ```

### 下面进行固件自定义，不推荐直接对.config进行编译，对小白来说很容易没选对依赖，我感觉没有虚拟/物理机好用。

1. 首先装好 Linux 系统，推荐 Debian 11 或 Ubuntu LTS

2. 安装编译依赖

   ```bash
   sudo apt update -y
   sudo apt full-upgrade -y
   sudo apt install -y ack antlr3 asciidoc autoconf automake autopoint binutils bison build-essential \
   bzip2 ccache cmake cpio curl device-tree-compiler fastjar flex gawk gettext gcc-multilib g++-multilib \
   git gperf haveged help2man intltool libc6-dev-i386 libelf-dev libglib2.0-dev libgmp3-dev libltdl-dev \
   libmpc-dev libmpfr-dev libncurses5-dev libncursesw5-dev libreadline-dev libssl-dev libtool lrzsz \
   mkisofs msmtp nano ninja-build p7zip p7zip-full patch pkgconf python2.7 python3 python3-pyelftools \
   libpython3-dev qemu-utils rsync scons squashfs-tools subversion swig texinfo uglifyjs upx-ucl unzip \
   vim wget xmlto xxd zlib1g-dev python3-setuptools
   ```

3. 下载你修改后的源代码，更新 feeds 并选择配置

   ```bash
   git clone https://github.com/yourid/lede
   cd lede
   ./scripts/feeds update -a
   ./scripts/feeds install -a
   make menuconfig
   ```

附上一份OpenWrt 编译 LuCI -> Applications 添加插件应用说明，https://www.right.com.cn/forum/thread-344825-1-1.html

4. 由于我们是利用Aciton进行云编译，所以在make menuconfig之后，只要把lede文件夹下的.config文件上传到Actions-OpenWrt上并覆盖原文件就行了（也可重命名，但重命名后要在build-openwrt.yml重新指定），之后就是常规的云编译步骤。
