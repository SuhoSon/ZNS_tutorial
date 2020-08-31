## nullblk를 통해 ZNS 생성
레포지토리에 포함된 `nullblk-zoned.sh` 를 실행하면 `/dev/nullb0` 디바이스가 생성됩니다.
``` bash
# ./nullblk-zoned.sh block_size zone_size number_of_conventional_zones number_of_sequential_zones
./nullblk-zoned.sh 4096 64 4 8
```

### ZNS 정보 출력
`/dev/nullb0`의 정보 출력을 위해선 ![ZNS tool](../ZNS-tools/ZNS-tools-isntall.md)을 통해 설치한 blkzone을 사용하시면 됩니다.
``` bash
./util-linux/blkzone report /dev/nullb0
```
