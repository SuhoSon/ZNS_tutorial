## blkzone을 위한 util-linux 설치
latest stable version 커널로 부팅 후, blkzone을 위해 util-linux를 설치합니다.
``` bash
git clone https://github.com/karelzak/util-linux.git
cd util-linux
apt install pkg-config autoconf autopoint libblkid-dev libtool -y
./autogen.sh
./configure
make -j *코어 수*
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

