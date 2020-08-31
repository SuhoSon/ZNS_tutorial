## ZNS에 ZoneFS 설치
nullb0에 ZoneFS를 설치(커널 5.6.0버전 이후부터 지원)하고, znsdir을 nullb0의 마운팅 포인트로 지정합니다.
정상적으로 ZoneFS가 설치되고 마운트 되었다면 znsdir 디렉토리엔 cnv 디렉토리와 seq 디렉토리가 생성됩니다.
``` bash
./zonefs-tools/src/mkzonefs /dev/nullb0
mkdir znsdir
./util-linux/mount --source=/dev/nullb0 --target=znsdir
ls znsdir
```

### Conventional zone에 ext4 filesystem 생성
znsdir/cnv 디렉토리에 있는 conventional zone file들은 일반 file 처럼 사용할 수 있습니다.
``` bash
apt install libmount-dev -y
mkfs.ext4 znsdir/cnv/0
mkdir cnvdir
./util-linux/mount --source=znsdir/cnv/0 --target=cnvdir
```

### Sequential zone에 dd로 write
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

dd if=/dev/zero of=znsdir/seq/0 oflag=direct bs=4k count=32

./util-linux/blkzone report /dev/nullb0
  start: 0x000000000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 0(nw) [type: 1(CONVENTIONAL)]
  start: 0x000020000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 0(nw) [type: 1(CONVENTIONAL)]
  start: 0x000040000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 0(nw) [type: 1(CONVENTIONAL)]
  start: 0x000060000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 0(nw) [type: 1(CONVENTIONAL)]
  **start: 0x000080000, len 0x020000, cap 0x020000, wptr 0x000100 reset:0 non-seq:0, zcond: 2(oi) [type: 2(SEQ_WRITE_REQUIRED)]**
  start: 0x0000a0000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x0000c0000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x0000e0000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x000100000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x000120000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x000140000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x000160000, len 0x020000, cap 0x020000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(em) [type: 2(SEQ_WRITE_REQUIRED)]
```

