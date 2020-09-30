---
layout: post
title:  "Python 메모리 누수 해결"
date:   2020-09-19 17:45:00 +0900
categories: dev
---

- Python 기반 AI 엔진 서비스를 GPU 서버에 배포하였는데 메모리가 지속적으로 증가하는 문제 발생
- 이를 해결하기 위해 memory_profiler, pympler 등을 사용하여 프로파일링 해보았지만 감지하지 못 함
- Python 기본 라이브러리 인 tracemalloc 을 사용하여 해결 하였음

<br/>

#### 참고 사이트

- <https://stackoverflow.com/questions/49991234/flask-app-memory-leak-caused-by-each-api-call>
- <https://www.fugue.co/blog/diagnosing-and-fixing-memory-leaks-in-python.html>
- <https://tech.buzzfeed.com/finding-and-fixing-memory-leaks-in-python-413ce4266e7d>

<br/>

#### 1. 이 전 메모리와 현재 메모리를 비교하여 크기 변화를 감지 하는 방식으로 matplotlib 문제 해결

##### (1) 삽입해야하는 코드
```python
import tracemalloc
tracemalloc.start()
my_snapshot = None

@app.route("/karyogram", methods=["POST"])
def karyogram():
    # 값을 계속 사용해야 하므로 전역 변수에 저장한다
    global my_snapshot
    if not my_snapshot:
        # 최초 메모리 상태를 저장한다
        my_snapshot = tracemalloc.take_snapshot()
    else:
        lines = []
        # 현재 메모리 상태를 최초와 비교하여 얼마나 차이가 나는지에 대한 통계를 구한다
        top_stats = tracemalloc.take_snapshot().compare_to(my_snapshot, 'lineno')
        # 메모리 사용량이 많은 순서대로 10개를 구하여 출력한다
        for stat in top_stats[:10]:
            lines.append(str(stat))
        print('\n'.join(lines), flush=True)
```

##### (2) 분석 결과
```bash
/usr/local/lib/python3.6/dist-packages/matplotlib/cbook/__init__.py:784: size=38.5 MiB (+25.6 MiB), count=6 (+4), average=6563 KiB
#/app/tools/tools.py:42: size=2514 KiB (-2154 KiB), count=41 (-7), average=61.3 KiB
#/usr/lib/python3.6/json/decoder.py:355: size=1342 KiB (+149 KiB), count=20258 (+1018), average=68 B
#/usr/local/lib/python3.6/dist-packages/numpy/core/shape_base.py:340: size=362 KiB (+137 KiB), count=31 (+11), average=11.7 KiB
#/usr/local/lib/python3.6/dist-packages/matplotlib/transforms.py:94: size=134 KiB (+89.5 KiB), count=1518 (+1016), average=90 B
#34.64.223.48 - - [31/Aug/2020 06:38:47] "POST /karyogram HTTP/1.1" 204 -

#Json Path: gclab/karyo/2020-03/20PB-369-PSH/slide0/cell7/auto-analized.json
#Image Saved
#----------------------
/usr/local/lib/python3.6/dist-packages/matplotlib/cbook/__init__.py:784: size=51.3 MiB (+38.5 MiB), count=8 (+6), average=6563 KiB
#/app/tools/tools.py:42: size=3591 KiB (-1077 KiB), count=46 (-2), average=78.1 KiB
#/usr/lib/python3.6/json/decoder.py:355: size=1011 KiB (-182 KiB), count=16058 (-3182), average=64 B
#/usr/local/lib/python3.6/dist-packages/matplotlib/transforms.py:94: size=178 KiB (+134 KiB), count=2026 (+1524), average=90 B
#/usr/local/lib/python3.6/dist-packages/matplotlib/lines.py:384: size=152 KiB (+116 KiB), count=71 (+54), average=2186 B
#34.64.223.48 - - [31/Aug/2020 06:39:00] "POST /karyogram HTTP/1.1" 204 -
```

- matplotlib 의 크기가 계속 증가하는 것을 파악
  - 분석 결과 2번째 줄 : **size=38.5 MiB (+25.6 MiB)**, count=6 (+4), average=6563 KiB
  - 분석 결과 12번째 줄 : **size=51.3 MiB (+38.5 MiB)**, count=8 (+6), average=6563 KiB
- `matplotlib memory leak` 을 키워드로 검색하여 `plt.close(fig)` 를 해줘야 한다는 것을 파악
  - <https://gist.github.com/astrofrog/824941#gistcomment-20976>
- `visualized.py` 의 `return_bytesio` 에 해당 코드를 추가하여 메모리 문제 해결

```python
# doai.engine.karyo.util.visualize 모듈
import matplotlib.pyplot as plt

def return_bytesio(image_array: np.array) -> BytesIO:
    img = BytesIO()
    fig = plt.figure(figsize=(12, 8))
    ax = plt.Axes(fig, [0.0, 0.0, 1.0, 1.0])
    ax.get_xaxis().set_visible(False)
    ax.get_yaxis().set_visible(False)
    ax.set_axis_off()
    fig.add_axes(ax)
    ax.imshow(image_array, aspect="auto")
    fig.savefig(img, bbox_inches="tight", pad_inches=0, format="png", dpi=130)
    img.seek(0)
    plt.close(fig) # 메모리 문제 해결을 위하여 새로 추가된 부분
    return img
```

<br/>

#### 2. 현재 메모리를 자세히 살피는 방법으로 new_karyogram 변수 메모리 문제 해결
- 첫 번째 방법 대로 변화를 구해 확인했지만 메모리가 크게 증가하는 부분을 찾지 못함
- 아래 방법으로 변화량 대신 현재 메모리를 좀 더 자세히 살펴보기로 함


##### (1) 삽입해야하는 코드
```python
@app.route("/infer", methods=["POST"])
def infer():
    # 현재 메모리 상태에 대한 통계를 생성하고 사용량이 많은 순서대로 5개 결과를 출력한다
    snapshot = tracemalloc.take_snapshot()
    for idx, stat in enumerate(snapshot.statistics('lineno')[:5], 1):
        print(str(stat), flush=True)
    # 메모리 사용량이 가장 많은 부분에 대한 정보를 상세히 출력한다
    traces = tracemalloc.take_snapshot().statistics('traceback')
    for stat in traces[:1]:
        print("memory_blocks=", stat.count, "size_kB=", stat.size / 1024, flush=True)
        for line in stat.traceback.format():
            print(line, flush=True)
```
##### (2) 분석 결과
```bash
/app/doai/engine/karyo/util/formatter.py:41: size=764 MiB, count=16707940, average=48 B
#<frozen importlib._bootstrap_external>:487: size=37.7 MiB, count=357314, average=111 B
#/usr/local/lib/python3.6/dist-packages/doai_module_storage/storage.py:38: size=5625 KiB, count=3, average=1875 KiB
#<frozen importlib._bootstrap>:219: size=4313 KiB, count=39221, average=113 B
#/usr/lib/python3.6/linecache.py:137: size=1313 KiB, count=12970, average=104 B

memory_blocks= 16707940 size_kB= 782641.4921875
  File "/app/doai/engine/karyo/util/formatter.py", line 41
    new_karyogram[key] = value.astype(int).tolist()
  File "app.py", line 70
    result = encode_to_json(result)
```

- `formatter.py` 에 `new_karyogram` 이 문제라는 것을 파악하여 해당 메소드 반복문 끝에 `del new_karyogram` 를 추가하여 해결하였음

```python
# doai.engine.karyo.util.formatter 모듈

def encode_to_json(result):
    encoded_result = []
    for karyogram in result:
        new_karyogram = dict()

        for key, value in karyogram.items():
            if isinstance(value, np.ndarray):
                new_karyogram[key] = value.astype(int).tolist()
            else:
                new_karyogram[key] = value

        new_karyogram["box"] = list(map(int, karyogram["box"]))
        encoded_result.append(new_karyogram)
        
        del new_karyogram # 메모리 문제 해결을 위해 추가된 부분
    return encoded_result
 ```