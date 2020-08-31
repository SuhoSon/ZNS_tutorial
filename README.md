# ZNS tutorial

<!--ts-->
   * [ZNS tutorial](#zns-tutorial)
      * [ZNS를 위한 커널 config 수정](#zns를-위한-커널-config-수정)
      * [ZNS를 emulate 하는 방법](#zns를-emulate-하는-방법)
      * [ZNS 관련 tool 설치](#zns-관련-tool-설치)
      * [ZNS에 ZoneFS 설치](#zns에-zonefs-설치)
      * [ZNS에 F2FS 설치](#zns에-f2fs-설치)
      * [libzbc를 기반으로 한 R/W 방법](#libzbc를-기반으로-한-rw-방법)

<!-- Added by: kijunking, at: Mon Aug 17 07:55:47 UTC 2020 -->

<!--te-->

## ZNS를 위한 커널 config 수정
ZNS를 사용하기 위해선 latest stable version의 커널을 사용하는 것이 추천됩니다.
.config 파일에서 아래 옵션들을 설정해 주시면 됩니다. (커널 5.7.14 버전 기준)
``` bash
CONFIG_BLK_DEV_ZONED=y
CONFIG_MQ_IOSCHED_DEADLINE=y
CONFIG_DM_ZONED=y
```

## ZNS를 emulate 하는 방법

![ZNS를 emulate 하는 방법 링크](./ZNS-emulate/ZNS-emulate.md)

## ZNS 관련 tool 설치

![ZNS 관련 tool 설치 링크](./ZNS-tools/ZNS-tools-install.md)

## ZNS에 ZoneFS 설치

![ZNS에 ZoneFS 설치 링크](./ZoneFS-tutorial/ZoneFS-install.md)

## ZNS에 F2FS 설치

![ZNS에 F2FS 설치 링크](./F2FS-tutorial/F2FS-install.md)

## libzbc를 기반으로 한 R/W 방법

![libzbc를 기반으로 한 R/W 방법 자료 링크](./libzbc-tutorial/libzbc-rw.md)

