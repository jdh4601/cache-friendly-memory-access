# cache-friendly memory access

목표: 캐시 최적화 & 메모리 접근 패턴 익히기

```bash
# compile하기
make
# 성능 테스트하기
./driver
```

## Rotate

### 1. Problem

`naive_rotate`함수에서 dst에 쓸 때 stride가 dim이라서 cache의 효율이 떨어진다.  
(spatial locality 부족으로 인한 cache miss 증가)

```c
char naive_rotate_descr[] = "naive_rotate: Naive baseline implementation";

void naive_rotate(int dim, pixel *src, pixel *dst)
{
    int i, j;

    for (i = 0; i < dim; i++)
	for (j = 0; j < dim; j++)
	    dst[RIDX(dim-1-j, i, dim)] = src[RIDX(i, j, dim)];
}
```

1. _src(i, j, dim)_
   내부 루프에서 i는 고정이고, j가 1씩 증가한다.
   (i, 0), (i, 1), (i, 2), ...
   가장 자연스러운 stride-1 접근 방법이다. 따라서 spatial_locality가 좋다.

2. _dst(dim-1-j, i, dim)_
   내부 루프에서 i는 고정이고, j가 1씩 증가한다.
   (dim-1, i), (dim-2, i), (dim-3, i), ...
   column-wise 접근 방법이라 spatial_locality가 매우 떨어지고 cache-hit가 매우 낮다.

### 2. Solution

```c
void rotate(int dim, pixel *src, pixel *dst)
{
    int i, j, ii, jj;
    int block_size = 16; // 16 또는 32

    // 1. 블록 단위 이동 (ii, jj)
    for (ii = 0; ii < dim; ii += block_size) {
        for (jj = 0; jj < dim; jj += block_size) {
            // 2. 블록 내부 픽셀 처리 (i, j)
            for (i = ii; i < ii + block_size; i++) {
                for (j = jj; j < jj + block_size; j++) {
                    dst[RIDX(dim-1-j, i, dim)] = src[RIDX(i, j, dim)];
                }
            }
        }
    }
}
```

_blocking 기법이란?_
이미지를 한 번에 처리하지 않고, 작은 블록 16x16 단위로 쪼개서 처리한다.  
그 블록 내의 데이터는 L1 cache안에 다 들어갈 수 있게 된다.  
덕분에 spatial locality 극대화된다.

blocking을 하는 이유는 캐시에 있는 데이터에 접근해야 하는데 column-wise방식은 spatial locality가 매우 떨어진다.

블록 크기: 16 x 16 x 12 bytes = 3KB
L1 cache: 32KB
이제 블록이 cache에 안정적으로 들어간다.!!

### 3. Ablation Study

block size 정하기

#### 1. block_size = 16일 때

```bash
Rotate: Version = rotate: Current working version:
Dim             64      128     256     512     1024    Mean
Your CPEs       2.9     3.0     4.5     4.8     8.4
Baseline CPEs   2.7     4.4     6.6     10.9    12.5
Speedup         0.9     1.5     1.5     2.3     1.5     1.5
```

CPE가 512x512에서 10.9 -> 4.8로 대폭 향상되었다.  
(참고로 CPE가 낮을수록 성능이 좋은 것이다.)

#### 2. block_size = 32일 때

```bash
Rotate: Version = rotate: Current working version:
Dim             64      128     256     512     1024    Mean
Your CPEs       2.7     2.9     4.2     6.0     13.8
Baseline CPEs   2.7     4.4     6.6     10.9    12.5
Speedup         1.0     1.5     1.6     1.8     0.9     1.3
```

block size를 16 -> 32로 늘렸더니 dim이 512부턴 더 성능이 저하되는 모습을 보인다.

### 4. Result

dim이 커지면 성능이 저하되는 이유.

#### 1. L1 cache 크기 제약

보통 L1 cache는 32KB정도다.
block size가 32이면, 32 x 32 x 4bytes = 4KB의 데이터를 다룬다.
하지만, rotate연산은 src 블록을 읽은 뒤, dst블록에 써야 한다.
즉, 두 개의 블록이 동시에 L1 cache 메모리 위에 올라와야 한다는 소리다!
4KB처럼 보이지만 최소 8KB가 필요하다.

#### 2. Conflict Miss

여기엔 캐시 구조(Set associativity) 문제가 있다.  
src, dst의 메모리 주소가 우연히 같은 set에 매핑되면 conflict가 발생한다.  
block size가 더 커질수록 더 높은 확률로 conflict가 발생한다.

따라서 16x16 block은 크기가 작아 conflict가 일어나도 피해가 적고, L1 cache에 안정적으로 들어간다. 그러나 32x32 block은 큰 dim=512인 이미지에서 cache 효율이 급격히 떨어진다.

### 5. Improvements

```c
# 포인터 연산
pixel *src_ptr = src + i * dim + k;      // 원본 시작점
pixel *dst_ptr = dst + (dim - 1 - k) * dim + i;  // 목적지 시작점

# loop unrolling
*dst_ptr++ = *src_ptr;
src_ptr += dim;  // 16번 반복
```

- RIDX 연산 제거 → 곱셈 제거 → 파이프라인 정체 감소
- src_row++, dst_col-=dim → prefetch가 잘 맞음
- loop unrolling으로 branch 감소
- block_size=16 블록 내에서 src/dst 모두 L1에 상주
