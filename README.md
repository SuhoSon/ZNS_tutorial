# ZNS tutorial

## ZNS를 위한 커널 config 수정
ZNS를 사용하기 위해선 latest stable version의 커널을 사용하는 것이 추천됩니다.
.config 파일에서 아래 옵션들을 설정해 주시면 됩니다. (커널 5.7.14 버전 기준)
``` bash
CONFIG_BLK_DEV_ZONED=y
CONFIG_MQ_IOSCHED_DEADLINE=y
CONFIG_DM_ZONED=y
```

## blkzone을 위한 util-linux 설치
latest stable version 커널로 부팅 후, blkzone을 위해 util-linux를 설치합니다.
``` bash
git clone https://github.com/karelzak/util-linux.git
cd util-linux
apt install autoconf autopoint libblkid-dev -y
./autogen.sh
./configure
make -j *코어 수*
```

## nullblk를 통해 ZNS 생성
다시 상위 디렉토리로 돌아와서 아래 명령어를 실행하면 /dev/nullb0 디바이스가 생성됩니다.
``` bash
# ./nullblk-zoned.sh block_size zone_size number_of_conventional_zones number_of_sequential_zones
./nullblk-zoned.sh 4096 64 4 8
```

## ZNS 정보 출력
nullb0의 정보 출력을 위해선 아래 명령어를 실행하면 됩니다.
``` bash
./util-linux/blkzone report /dev/nullb0
```

## mkzonefs tool 설치
ZoneFS를 설치하기위해 mkzonefs 툴을 다운로드하고 컴파일합니다.
``` bash
git clone https://github.com/damien-lemoal/zonefs-tools.git
cd zonefs-tools
./autogen.sh
./configure
make -j *코어 수*
```

## ZNS에 ZoneFS 설치
다시 상위 디렉토리로 돌아와서,
nullb0에 ZoneFS를 설치(커널 5.6.0버전 이후부터 지원)하고, znsdir을 nullb0의 마운팅 포인트로 지정합니다.
정상적으로 ZoneFS가 설치되고 마운트 되었다면 znsdir 디렉토리엔 cnv 디렉토리와 seq 디렉토리가 생성됩니다.
``` bash
./zonefs-tools/mkzonefs /dev/nullb0
mkdir znsdir
./util-linux/mount --source=/dev/nullb0 --target=znsdir
ls znsdir
```

## Conventional zone에 ext4 filesystem 생성
znsdir/cnv 디렉토리에 있는 conventional zone file들은 일반 file 처럼 사용할 수 있습니다.
``` bash
apt install libmount-dev -y
mkfs.ext4 znsdir/cnv/0
mkdir cnvdir
./util-linux/mount --source=znsdir/cnv/0 --target=cnvdir
```

## Sequential zone에 dd로 write
znsdir/seq 디렉토리에 있는 sequential zone file들에 write하면 write pointer가 변하는 것을 볼 수 있습니다.
``` bash
./util-linux/blkzone report /dev/nullb0
  start: 0x000000000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 0(nw) [type: 1(CONVENTIONAL)]
  start: 0x000020000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 0(nw) [type: 1(CONVENTIONAL)]
  start: 0x000040000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 0(nw) [type: 1(CONVENTIONAL)]
  start: 0x000060000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 0(nw) [type: 1(CONVENTIONAL)]
  **start: 0x000080000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]**
  start: 0x0000a0000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x0000c0000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x0000e0000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x000100000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x000120000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x000140000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x000160000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]

dd if=/dev/zero of=znsdir/seq/0 bs=4096 count=1 conv=notrunc oflag=direct
./util-linux/blkzone report /dev/nullb0
  start: 0x000000000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 0(nw) [type: 1(CONVENTIONAL)]
  start: 0x000020000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 0(nw) [type: 1(CONVENTIONAL)]
  start: 0x000040000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 0(nw) [type: 1(CONVENTIONAL)]
  start: 0x000060000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 0(nw) [type: 1(CONVENTIONAL)]
  **start: 0x000080000, len 0x020000, cap 0x020000, wptr 0x000008 reset:0 non-seq:0, zcond: 2(oi) [type: 2(SEQ_WRITE_REQUIRED)]**
  start: 0x0000a0000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x0000c0000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x0000e0000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x000100000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x000120000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x000140000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x000160000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
```

## ZNS에 write 하는 방법
``` bash
TODO
```
