# libzbc를 통한 R/W 방법

<!--ts-->
   * [libzbc를 통한 R/W 방법](#libzbc를-통한-rw-방법)
      * [들어가기 전에](#들어가기-전에)
      * [프로그램 작성 기초](#프로그램-작성-기초)
      * [Zone을 가져오기](#zone을-가져오기)
      * [버퍼의 준비](#버퍼의-준비)
      * [쓰기 및 읽기의 시작](#쓰기-및-읽기의-시작)
         * [쓰기](#쓰기)
         * [읽기](#읽기)
      * [메모리 해제](#메모리-해제)
      * [소스 코드 전체](#소스-코드-전체)

<!-- Added by: kijunking, at: Wed Aug 19 10:53:21 UTC 2020 -->

<!--te-->

## 들어가기 전에

[libzbc](https://github.com/hgst/libzbc)와 Zoned Storage가 확보되었고,
그 디바이스를 가리키는 폴더는 `/dev/sda`라고 하겠습니다.

그리고 반드시 [libzbc/documentation](https://github.com/hgst/libzbc/tree/master/documentation)
내용을 `doxygen`을 통해서 각각의 함수에 대한 상세한 역할이 무엇인지에 관한 자료를 확보해주시길 바랍니다.
README.md의 내용도 충분히 좋긴 하나, 정확한 매개 변수의 입출력에 관해서 알기 위해서는 해당 자료가 반드시 필요합니다.

## 프로그램 작성 기초

libzbc를 사용하기 위한 가장 기본적인 뼈대는 아래와 같이 만들어줍니다.
해당 파일의 이름은 `sample.c`라고 가정하겠습니다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <libzbc/zbc.h>

#define ZBC_PATH "/dev/sda"

int main(void) {
    struct zbc_device *dev = NULL;
    struct zbc_device_info info = { 0 };
    int ret = 0;

    if (0 > (ret = zbc_open(ZBC_PATH, O_RDWR, &dev)) {
        perror("zbc open error");
        return -1;
    }

    zbc_get_device_info(dev, &info);
    zbc_print_device_info(&info, stdout);

    /* 입출력 관련 부분 */

    zbc_close(dev);
    return 0;
}

```

위에서 보이는 바와 같이 작성을 한 후에 아래의 명령을 통해서 빌드를 진행합니다.

```bash
gcc sample.c -lzbc
```

## Zone을 가져오기

그리고 실행을 하게 되면 현재 Zoned Storage `/dev/sda`에 대한 정보가 나오게 됩니다.
이제 위 코드에서 `입출력 관련 부분`에 R/W 명령을 수행해보도록 하겠습니다.

가장 먼저 해야 할 일은 적절한 Zone을 가져오는 일입니다.
이를 하기 위해서는 `zbc_list_zones()` 함수를 사용하면 됩니다.

해당 함수에서 가져오는 Zone을 원하는 특성에 맞게 플래그를 줘서 추출할 수 있습니다.
주요 플래그는 아래와 같으며, RO는 Reporting Order를 의미합니다.

| 항  목 | 내  용 |
|:------:|:-------|
|`ZBC_RO_ALL` | 장치 안에 있는 모든 Zone들을 가져오라는 것을 의미합니다. (Conventional Zone도 가져오므로 유의해주시길 바랍니다.) |
|`ZBC_RO_EMPTY` | Empty 상태의 Zone을 가져옵니다. |
|`ZBC_RO_IMP_OPEN` | Implicit Open 상태의 Zone을 가져옵니다. |
|`ZBC_RO_EXP_OPEN` | Explicit Open 상태의 Zone을 가져옵니다. |
|`ZBC_RO_CLOSED` | Close 상태의 Zone을 가져옵니다. |
|`ZBC_RO_FULL` | Full 상태의 Zone을 가져옵니다.|
|`ZBC_RO_RDONLY` | Read Only 상태의 Zone을 가져옵니다. |
|`ZBC_RO_OFFLINE` | Offline 상태의 Zone을 가져옵니다. |
|`ZBC_RO_RWP_RECOMMENDED` | Reset write pointer recommended 상태의 Zone을 가져옵니다. |
|`ZBC_RO_NON_SEQ` | Host aware 장치에서 무작위 쓰기가 벌어진 Zone을 가져옵니다 |
|`ZBC_RO_NOT_WP` | Conventional Zone을 가져옵니다.|
|`ZBC_RO_PARTIAL` | 이것은 필터가 아니라 Command Buffer 크기만큼만 Zone을 가져옴을 의미합니다. |

상태에 대한 자세한 내용은 [링크](http://www.t13.org/Documents/UploadedDocuments/docs2015/di537r05-Zoned_Device_ATA_Command_Set_ZAC.pdf)의
자료를 참고해주시길 바랍니다.

이를 테면, 새롭게 데이터를 쓴다고 가정하는 경우에는 다음과 같이 진행하면 됩니다.

```c
struct zbc_device *dev = NULL; // 디바이스 정보가 있습니다.
struct zbc_zone *zone_list = NULL; // Zone 리스트가 들어갑니다.
struct zbc_zone *zone = NULL;
int nr_zones; // Zone의 갯수가 들어갑니다.

/* device를 설정하는 부분 생략 */

ret = zbc_list_zones(dev, 0, ZBC_RO_EMPTY, &zone_list, &nr_zones);
if (ret < 0) {
    errno = -ret;
    perror("zbc_list_zones failed");
    return -1;
}

zone = &zone_list[0];
if (!zbc_zone_sequential(zone)) {
    errno = EINVAL;
    perror("invalid or cannot find zone");
    return -1;
}
```

`zbc_list_zones()`에서 0은 탐색 시작 지점(sector)에 해당합니다.
이는 처음부터 읽음을 의미합니다.

만약에 임의의 쓰기 명령을 날리게 되면 `zbc_zone_operation()`을 통해
Explicit Open 명령어를 날리지 않는 이상은 Implicit Open 상태이므로
다음 번에 해당 존을 찾을 때에는 `ZBC_RO_IMP_OPEN`을 쓰면 됩니다.

탐색이 완료되었으면 탐색된 갯수가 `nr_zones`를 통해서 확인해보도록 해주어야 합니다.
본 코드에서는 해당 부분은 생략되었습니다.

그리고 우리는 Empty Zone의 가장 처음 Zone을 가져오고 해당 Zone이
Conventional Zone이 아님을 확인하도록 합니다.

## 버퍼의 준비

Zone에 쓰기 요청을 하기 위해서 메모리를 할당을 받을 때에는 아래와 같이 특수한
방식으로 받아야 합니다.

```c
void *buffer = NULL;
size_t size;
size_t count;

size = sysconf(_SC_PAGESIZE);

ret = posix_memalign((void **)&buffer, sysconf(_SC_PAGESIZE), size);
if (ret < 0) {
    fprintf(stderr, "Memory allocation failed...\n");
    return -1;
}
sprintf(buffer, "Hello World\n");

count = size >> 9; // Sector 크기가 512 bytes 이므로
```

버퍼의 크기는 자유롭게 해도 괜찮으나 되도록 메모리 페이지 크기의 배수로 잡아줘야 합니다.
현재는 해당 크기의 1배로 지정했습니다.

이렇게 지정해주는 이유는 버퍼를 확보하면서 메모리가 연속적일 수 있도록
align을 메모리의 페이지 크기로 설정해줘야 하기 때문입니다.

만든 버퍼에 대해서 Zone의 최소 단위인 512 bytes(2^9)로 나누어 Sector 몇 개에 쓸 지
갯수(`count`)를 확보합니다.

## 쓰기 및 읽기의 시작

### 쓰기

메모리 확보가 완료가 되었으면 이제는 쓰기를 할 차례입니다. 소스 코드는 아래와 같이 작성해주시면 됩니다.

```c
// 현재의 Write Pointer를 가져오도록 합니다.
offset = zbc_zone_wp(target_zone);
printf("offset(%d), count(%d)\n", offset, count);

// buffer에 대해서 offset 위치에 512 * count만큼 buffer 내용을 write 합니다.
ret = zbc_pwrite(dev, buffer, count, offset);
if (ret <= 0) {
    errno = -ret;
    perror("Fail to run pwrite");
    return -1;
}
```

먼저, 쓰기 위치를 확보하기 위해서 Zone의 Write Pointer(WP)를 가져오도록 합니다.
쓰기는 log-structured이기 때문에 앞으로만 나아가게 됩니다.

그리고 아까 확보한 `buffer`의 `count` 값을 가지고 `zbc_pwrite()`를 수행해줍니다.
이를 통해서 쓰기를 할 수 있습니다.

### 읽기

읽기를 하는 경우에는 동일한 Zone을 확보한 뒤에 `zbc_zone_start(zone)`에서
내가 읽고 싶은 위치로 이동하면 됩니다.

소스 코드로는 아래와 같습니다.

```c
idx = 0;
/**
 * - dev: device를 나타냅니다.
 * - buffer: read한 내용이 담기는 위치에 해당합니다.
 * - count: 512 * count 만큼 가져오도록 합니다.
 * - zbc_zone_start(zone): zone에 해당하는 시작 위치를 가져옵니다.
 */
ret = zbc_pread(dev, buffer, count, zbc_zone_start(zone) + count * idx);
if (ret <= 0) {
        errno = -ret;
        perror("Fail to run pread");
        return -1;
}
printf("%s\n", buffer);
```

`zbc_pwrite`랑 거의 동일하게 사용합니다.
다만, `zbc_zone_start(zone) + count * idx`가 있습니다.
이는 현재 buffer의 크기가 512MB의 count배이므로 원하는 위치를 찾기 위해
다음 버퍼로 이동해주기 위해서 작성하게 되었습니다.

## 메모리 해제

해제는 일반적인 해제와 거의 유사합니다.

```c
zbc_close(dev);
free(buffer);
free(zone_list);
```

## 소스 코드 전체

결과적으로, 앞에서 설명한 내용을 종합하면 아래와 같은 코드가 나옴을 확인할 수 있었습니다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <libzbc/zbc.h>

#define ZBC_PATH "/dev/sda"
#define READ_POSITION 2

#define DO_WRITE
#define DO_READ

int main(void)
{
        struct zbc_device *dev = NULL;
        struct zbc_device_info info = { 0 }; /**< information 저장용 */

        struct zbc_zone *zone_list = NULL;
        struct zbc_zone *zone = NULL;

        int nr_zone_list;
        int ret;
        int offset;

        void *buffer = NULL;
        size_t size = 0, count = 0;
        int idx = 0;

        ret = zbc_open(ZBC_PATH, O_RDWR, &dev);
        if (0 > ret) {
                perror("zbc open error");
                return -1;
        }
        printf("ret: %d, dev: %p\n", ret, dev);

        // device 정보를 가져오는 부분에 해당합니다.
        zbc_get_device_info(dev, &info);

        // device의 정보를 가져옵니다.
        zbc_print_device_info(&info, stdout);

        /**
         * - zone_list에 zone들에 대한 정보가 들어가게 됩니다.
         * - RO의 의미는 REPORT_OPTION의 줄임말입니다.
         * - ZBC_RO_ALL을 하면 Conventional Disk도 들어갈 수 있으므로 주의해주시길 바랍니다.
         */
        ret = zbc_list_zones(dev, 0, ZBC_RO_IMP_OPEN, &zone_list, &nr_zone_list);
        if (ret < 0) {
                errno = -ret;
                perror("zbc_list_zones failed");
                return -1;
        }
        printf("get zone list %d\n", nr_zone_list);

        /**
         * - Sequential Zone을 가져오고 Sequential Zone이 맞는 지 확인하도록 합니다.
         */
        zone = &zone_list[0];
        if (!zbc_zone_sequential(zone)) {
                errno = EINVAL;
                perror("invalid zone");
                return -1;
        }

        size = sysconf(_SC_PAGESIZE); // 버퍼의 크기를 페이지 크기로 설정합니다.

        // 메모리를 정렬을 해주도록 합니다.
        ret = posix_memalign((void **) &buffer, sysconf(_SC_PAGESIZE), size);
        if (ret < 0) {
                fprintf(stderr, "Memory allocation failed...\n");
                return -1;
        }
        sprintf(buffer, "Next2\n");

        /**
         * sector가 512 bytes 단위이므로 2^9으로 size를 나누도록 합니다.
         * Zone의 크기가 변경된다면 이 값도 변경되어야 합니다.
         */
        count = size >> 9;

#ifdef DO_WRITE
        // 현재의 Write Pointer를 가져오도록 합니다.
        offset = zbc_zone_wp(zone);
        printf("offset(%d), count(%d)\n", offset, count);

        // buffer에 대해서 offset 위치에 512 * count만큼 buffer 내용을 write 합니다.
        ret = zbc_pwrite(dev, buffer, count, offset);
        if (ret <= 0) {
                errno = -ret;
                perror("Fail to run pwrite");
                return -1;
        }
#endif

        sprintf(buffer, "\n");

#ifdef DO_READ
        idx = READ_POSITION;
        /**
         * - dev: device를 나타냅니다.
         * - buffer: read한 내용이 담기는 위치에 해당합니다.
         * - count: 512 * count 만큼 가져오도록 합니다.
         * - zbc_zone_start(zone): zone에 해당하는 시작 위치를 가져옵니다.
         */
        ret = zbc_pread(dev, buffer, count, zbc_zone_start(zone) + count * idx);
        if (ret <= 0) {
                errno = -ret;
                perror("Fail to run pread");
                return -1;
        }
        printf("%s\n", buffer);
#endif

        zbc_close(dev);
        free(buffer);
        free(zone_list);

        return 0;
}
```

