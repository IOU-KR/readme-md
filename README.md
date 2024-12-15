## Q. 최근에 자신이 성장했다고 느낀 순간과 그 계기를 구체적으로 쓰시오.

3D프린팅을 취미로 하고 있었다. 내 3D프린터에는 멀티컬러 출력기능이 있다. 여기서 멀티컬러 출력이란 여러가지 색의 필라멘트를 한 출력물에 동시에 사용해서 단색이 아닌 출력물을 뽑아내는 FDM 3D프린팅의 기술이다. 내 3D프린터는 노즐이 여러개가 아니기 때문에 여러 색상의 필라멘트를 하나의 노즐에서 사용해야한다. 멀티 컬러 출력에는 컬러 채인지라는 과정이 필요한데 말 그대로 색상을 바꾸는 작업이다.
하나의 노즐에서 색상을 바꾸면 기존에 사용하던 색상의 필라멘트가 200도 액체상태로 노즐에 소량 남게 되는데 이를 바꾸고자 하는 색상의 필라멘트로 밀어내는 **퍼징**(purging)이 필요하다. 그 소량이 생각보다 치명적이라 퍼징이 적으면 색상이 섞여 출력물이
깨끗하지 못하고 너무 많으면 재료가 낭비된다.  
적절한 퍼징 값을 찾기 위해서는 퍼징 켈리브레이션 모델을 3D프린터로 출력해야 하는데 여기서 문제가 발생했다.
내 3D프린터는 각 출력마다 출력 준비 시간이 있는데 이것이 7분 정도로 모델을 출력하는 시간보다 더 길기 때문이다.  
따라서 나는 효율적으로 여러 켈리브레이션 모델을 한 출력 파일안에 넣는 알고리즘을 연구하기 시작했다.


(켈리브레이션 모델을 여러번 뽑아야하는 이유는 n가지 색상을 사용한다고 할때 색상 변경 경우의 수가 <a>n</a><a style="font-size: 25px;">P</a><a>2</a> 이기 때문이다.)

---
### 참고그림
퍼징 켈리브레이션 모델은 아래와 같이 생겼다.

<img src="https://makerworld.bblmw.com/makerworld/model/US35d02d5dbfd7b1/162143248/instance/2024-11-24_6173598e117fc.png?x-oss-process=image/resize,w_1920/format,webp" width="400px">

왼쪽은 `검정 -> 하양`

오른쪽은 `하양 -> 검정`

---

4개의 색상을 켈리브레이션한다면 다음과 같이 출력 파일을 구성할 수 있다.

- 파일 1 (변경 순서: `1 -> 2 -> 3 -> 4`)
  - `1 -> 2`
  - `2 -> 3`
  - `3 -> 4`
- 파일 2 (변경 순서: `2 -> 4 -> 1 -> 3`)
  - `2 -> 4`
  - `4 -> 1`
  - `1 -> 3`
- 파일 3 (변경 순서: `3 -> 1 -> 4 -> 2`)
  - `3 -> 1`
  - `1 -> 4`
  - `4 -> 2`
- 파일 4 (변경 순서: `4 -> 3 -> 2 -> 1`)
  - `4 -> 3`
  - `3 -> 2`
  - `2 -> 1`

이때 중요한 것은 파일마다 절대적인 변경 순서가 존재한다는 것이다.  
`1 -> 2 -> 1 -> 4`와 같은 중복되는 변경 순서는 존재할 수 없고 따라서 `1 -> 2`, `2 -> 1`은 같은 파일에 들어올 수 없다.  
여기서 나는 `1 -> 2`, `2 -> 3`, `3 -> 4`와 같은 퍼징 모델의 묶음을 `1 -> 2 -> 3 -> 4`와 같은 변경 순서로도 나타낼 수 있다는 사실을 발견했다.

이어서 출력 색상의 개수를 n이라 할때, n이 짝수면 한 출력 파일에 n-1개의 퍼징 모델을 넣고 n개의 출력 파일로 나눌 수 있음을 알았다.  
n=4의 경우 다음과 같이 표현 가능하다.

- `1 -> 2 -> 3 -> 4`
- `2 -> 4 -> 1 -> 3`
- `3 -> 1 -> 4 -> 2`
- `4 -> 3 -> 2 -> 1`

n이 홀수인 경우엔 달랐는데, 한 출력 파일에 n-1개의 퍼징 모델을 넣는 다면 최대 n-1개의 출력 파일로 내눌 수 있고 약간의 퍼징 모델이 남을을 보았다.  
n=3의 경우 다음과 같이 표현 가능하다.

- `1 -> 2 -> 3`
- `3 -> 2 -> 1`
- 나머지: `1 -> 3`, `3 -> 1`

여기서 나는 이 문제를 일단 이렇게 정리했다.

---

### 정의

1. **자식 순열**이란 순열을 끊었을때 나오는 순열로 정의한다.  
(예: 순열 `[1]-[2]`은 순열 `[1]-[2]-[3]-[4]`의 첫번째 자식 순열이다.)

2. **자식 순열 집합**이란 주어진 순열의 모든 자식 순열의 집합으로 정의한다.

3. "**순열을 끊는다**"는 순열의 순서를 알아볼 수 있도록 최소단위로 잘라내는 것으로 정의한다.  
(예: `[1]-[2]-[3]-[4]` -> `[1]-[2]` or `[2]-[3]` or `[3]-[4]` or `[4]-[5]`)

### 문제 설명

n개의 서로다른 카드가 있다. (2<=n이고 n은 자연수다.)  
이 카드들은 각각 1부터 n까지의 자연수를 번호로 가지고 있다.  
예시로 n=4라면 상황을 아래와 같이 표현가능하다.

    [1] [2] [3] [4]

여기서 우리는 아래 **조건 4개**를 모두 만족하는 집합 A를 구하는 효울적인 알고리즘을 찾아야 한다.  
(부르트포스는 되도록 제외한다.)

#### 1. A의 원소는 모두 주어진 카드에 대한 순열이다.
#### 2. 카드를 카드번호에 대한 오름차순으로 나열한 순열은 반드시 A에 포함된다.
예시로 n=4라면, `[1]-[2]-[3]-[4]`가 A에 포함된다.
#### 3. 임의의 순열을 A에서 뽑고 각각 순열1, 순열2라하자. 그러면 순열1의 자식 순열 집합과 순열 2의 자식 순열 집합의 교집합은 공집합이다.
#### 4. A는 위 조건을 만족하는 집합들 중 가장 원소의 개수가 많아야한다.
만약에 조건 4개를 만족하는 A의 수가 1이상이면 그 중 아무거나 하나만 반환해도 된다.

처음에 내가 만든 이 문재를 보고 나도 당황했다. 지금봤을때 이 선택이 좋았는지는 모르지만 나는 퍼징 모델을 조합해서 푸는 방식이 아니라 변경 순서의 집합을 구하는 방식으로 접근했다. 무언가 규칙을 찾을 수 있을거라 생각했기 때문이다.  
수학에서 배운 순열을 바탕으로 내가 만약 이 문제에 부르트포스를 사용한다면 검토해야할 데이터의 양이 `n!`이 된다. 이는 n의 값이 커질 수록 매우 빠르게 부풀기 떄문에 비효율적이다. 그래서 나는 혹시나 하는 마음으로 수학을 잘하시는 담임 선생님께 이것의 규칙성에 대해 아는 것이 있는지 물어보기로 했다. 선생님도 이 문제는 처음본다 하셨으며 나의 철번째 알고리즘과 비슷한 제안을 하셨다.

> "카드의 가능한 모든 순열들을 구한다.  
> 순열들 중에서 3번 조건에 만족하는 것들을 뽑는다.  
> 그 중에서 아무거나 뽑으면서 모든 조건을 만족하는  
> 집합 A를 만들 수 있는지 확인하는 것을 제귀적으로 반복한다."

여기서 제귀가 문제였다. 내가 지금까지 구현한 제귀 함수들 중에서 가장 복잡했기 때문이다. 코드가 복잡해 디버깅도 어려울 것이고 코딩 시간도 오래걸릴 것이였다. 그래서 나는 일단 집합 A에 무언가 간단한 규칙성이 있다는 가정하에 부르트포스로 A를 구해보았다.  
부르트포스로는 n=6까지 구할 수 있었고 n=7부터는 메모리 부족으로 프로세스가 죽거나 너무 느려서 결과를 받을 수 없었다.
난 이 한정된 자료들 속에서 추측과 검증을 반복하며 아래와 같은 규칙을 발견했다.

**n이 짝수**라면 다음과 같은 규칙을 따르는 A가 무조건 존재한다.

|||||
|:-:|:-:|:-:|:-:|
|1|2|...|`n`|
|2|*?*|*?*|`n-1`|
|...|*?*|*?*|...|
|`n`|`n-1`|...|1|
|||||

예를 들어 **n=6**라면

|||||||
|:-:|:-:|:-:|:-:|:-:|:-:|
|1|2|3|4|5|6|
|2|*?*|*?*|*?*|*?*|5|
|3|*?*|*?*|*?*|*?*|4|
|4|*?*|*?*|*?*|*?*|3|
|5|*?*|*?*|*?*|*?*|2|
|6|5|4|3|2|1|
|||||||

**n이 홀수**라면 다음과 같은 규칙을 따르는 A가 무조건 존재한다.

|||||||
|:-:|:-:|:-:|:-:|:-:|:-:|
|1|2|...|`(n+1)/2`|...|`n`|
|2|*?*|*?*|`(n+1)/2`|*?*|`n-1`|
|...|||||...|
|`(n+1)/2-1`|*?*|*?*|`(n+1)/2`|*?*|`(n+1)/2+1`|
|`(n+1)/2+1`|*?*|*?*|`(n+1)/2`|*?*|`(n+1)/2-1`|
|...|||||..|
|`n`|`n-1`|...|`(n+1)/2`|...|1|
|||||||

예를 들어 **n=7**라면

||||||||
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|1|2|3|4|5|6|7|
|2|*?*|*?*|4|*?*|*?*|6|
|3|*?*|*?*|4|*?*|*?*|5|
|5|*?*|*?*|4|*?*|*?*|3|
|6|*?*|*?*|4|*?*|*?*|2|
|7|6|5|4|3|2|1|
||||||||

이제 나는 저 조건을 만족하는 집합 A를 구하는 부르트포스를 작성할 수 있고 경우의 수를 획기적으로 줄일 수 있고 그 결과 n=8
까지 계산 성공했다,

난 여기서 한 번 더 생각했다. 저렇게 규칙이 정해졌으니 사실상 우리가 해야할 것은 저 물음표속에 숫자를 집어넣기만 하면된다.  
그래서 난 문제의 뿌리로 돌아왔고 빈칸에 숫자를 채워넣는 방법을 생각했다. 그리고 여기서 아주 중요한 규칙성 2가지를 발견했다.  
예를 들어 n=4라면 아래와 같이 표현가능한데
|||||
|:-:|:-:|:-:|:-:|
|1|2|3|4|
|2|4|1|3|
|3|1|4|2|
|4|3|2|1|
|||||

이를 위 아래 절반으로 나누어 보자.

|||||
|:-:|:-:|:-:|:-:|
|1|2|3|4|
|2|4|1|3|
|||||

|||||
|:-:|:-:|:-:|:-:|
|3|1|4|2|
|4|3|2|1|
|||||

여기서 하나를 180도 회전하면

|||||
|:-:|:-:|:-:|:-:|
|1|2|3|4|
|2|4|1|3|
|||||

|||||
|:-:|:-:|:-:|:-:|
|1|2|3|4|
|2|4|1|3|
|||||

둘은 일치한다.  
이 페턴은 n에 관계없이 항상 나타난다.

그리고 n=4를 시계방향으로 45도 돌려보자.
| | | | | | | |
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| | | |1| | | |
| | |2| |2| | |
| |3| |4| |3| |
|4| |1| |1| |4|
| |3| |4| |3| |
| | |2| |2| | |
| | | |1| | | |

그럼 이 도형은 좌우 대칭이고 상하 대칭이다.  
이 패턴은 n이 짝수일때만 나타나는데 홀수까지 확장하기 위해서 홀수 도형을 살짝 수정했다.

|||||||
|:-:|:-:|:-:|:-:|:-:|:-:|
|1|2|...|`(n+1)/2`|...|`n`|
|2|*?*|*?*|`(n+1)/2`|*?*|`n-1`|
|...|||||...|
|`(n+1)/2`|`(n+1)/2`|`(n+1)/2`|`(n+1)/2`|`(n+1)/2`|`(n+1)/2`|
|...|||||...|
|`n`|`n-1`|...|`(n+1)/2`|...|1|
|||||||

예를 들어 **n=7**라면

||||||||
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|1|2|3|4|5|6|7|
|2|*?*|*?*|4|*?*|*?*|6|
|3|*?*|*?*|4|*?*|*?*|5|
|4|4|4|4|4|4|4|
|5|*?*|*?*|4|*?*|*?*|3|
|6|*?*|*?*|4|*?*|*?*|2|
|7|6|5|4|3|2|1|
||||||||

이렇게 도형을 수정했고 이제 위 규칙을 모두 적용할 수 있다.

나는 그렇게 3번째 알고리즘을 작성했다.
> "위 모든 규칙을 따르는 A를 구하기 위해 A를 2차원 배열로 표현하고 규칙들을 구현한다."  
> "예를 들어 **n=4**라면 초기 A는 아래와 같이 생겼다."

||||||||
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|1|2|3|4|5|6|7|
|2|*?*|*?*|4|*?*|*?*|6|
|3|*?*|*?*|4|*?*|*?*|5|
|4|4|4|4|4|4|4|
|5|*?*|*?*|4|*?*|*?*|3|
|6|*?*|*?*|4|*?*|*?*|2|
|7|6|5|4|3|2|1|
||||||||

> "각 빈자리에 올 수 있는 숫자들을 문제의 조건을 이용해 모두 구한다."

||||||||
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|1|2|3|4|5|6|7|
|2|`{5,7}`|`{1,7}`|4|`{1,6,7}`|`{1,3}`|6|
|3|*?*|*?*|4|*?*|*?*|5|
|4|4|4|4|4|4|4|
|5|*?*|*?*|4|*?*|*?*|3|
|6|*?*|*?*|4|*?*|*?*|2|
|7|6|5|4|3|2|1|
||||||||

> "가장 작은 숫자로 순서대로 채워 넣는다. 만약 빈자리에 올 수 있는 숫자가 하나밖에 없으면 그것부터 처리한다."

||||||||
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|1|2|3|4|5|6|7|
|2|**5**|`{1,7}`|4|`{1,6,7}`|`{1,3}`|6|
|3|*?*|*?*|4|*?*|*?*|5|
|4|4|4|4|4|4|4|
|5|*?*|*?*|4|*?*|*?*|3|
|6|*?*|*?*|4|*?*|**5**|2|
|7|6|5|4|3|2|1|
||||||||

||||||||
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|1|2|3|4|5|6|7|
|2|**5**|**1**|4|`{6,7}`|`{3}`|6|
|3|**1**|*?*|4|*?*|*?*|5|
|4|4|4|4|4|4|4|
|5|*?*|*?*|4|*?*|**1**|3|
|6|*?*|*?*|4|**1**|**5**|2|
|7|6|5|4|3|2|1|
||||||||

||||||||
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|1|2|3|4|5|6|7|
|2|**5**|**1**|4|`{6,7}`|**3**|6|
|3|**1**|*?*|4|*?*|*?*|5|
|4|4|4|4|4|4|4|
|5|*?*|*?*|4|*?*|**1**|3|
|6|**3**|*?*|4|**1**|**5**|2|
|7|6|5|4|3|2|1|
||||||||

||||||||
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|1|2|3|4|5|6|7|
|2|**5**|**1**|4|**6**|**3**|6|
|3|**1**|*?*|4|*?*|**6**|5|
|4|4|4|4|4|4|4|
|5|**6**|*?*|4|*?*|**1**|3|
|6|**3**|**6**|4|**1**|**5**|2|
|7|6|5|4|3|2|1|
||||||||

> "이를 반복한다."

||||||||
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|1|2|3|4|5|6|7|
|2|5|1|4|6|3|6|
|3|1|`{7}`|4|`{2}`|6|5|
|4|4|4|4|4|4|4|
|5|6|*?*|4|*?*|1|3|
|6|3|6|4|1|5|2|
|7|6|5|4|3|2|1|
||||||||

||||||||
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|1|2|3|4|5|6|7|
|2|5|1|4|6|3|6|
|3|1|**7**|4|**2**|6|5|
|4|4|4|4|4|4|4|
|5|6|**2**|4|**7**|1|3|
|6|3|6|4|1|5|2|
|7|6|5|4|3|2|1|
||||||||

사실 이 알고리즘은 약간의 추측이 들어갔다.

**"가장 작은 숫자로 순서대로 채워 넣는다. 만약 빈자리에 올 수 있는 숫자가 하나밖에 없으면 그것부터 처리한다."**

이 말대로 했을때 정말 오류없이 완성되는가 였다.  
나는 일단 오류 없이 완성될거라 추측하고 프로그램을 작성했다. 그 결과는 성공이였다. 검증되진 않았지만 n=16을 넘어 n=200까지도 계산이 가능했으며 오류 없이 동작함을 보았다.

이제 난 욕심이 생기기 시작했다. 이미 목표를 달성했지만 더 강력한 알고리즘을 만들고 싶었다.
그래서 완성된 도형들의 모양을 n=4부터 n=8까지 관찰했다. 그리고 다음과 같은 가장 중요한 마지막 규칙을 발견했다.

아래 n=4를 예로들자.
|||||
|:-:|:-:|:-:|:-:|
|1|2|3|4|
|2|*4*|*1*|3|
|3|*1*|*4*|2|
|4|3|2|1|
|||||

위아래가 같으니 반으로 잘라보자.
|||||
|:-:|:-:|:-:|:-:|
|1|2|3|4|
|2|*4*|*1*|3|
|||||

여기서 비스듬하게 적힌 숫자가 알고리즘이 구한 숫자인데 이는 이렇게 이전 배열에서 구할 수 있다.

|||||
|:-:|:-:|:-:|:-:|
|1|2|3|`4`|
|2|`4`|*?*|3|
|||||

|||||
|:-:|:-:|:-:|:-:|
|`1`|2|3|4|
|2|4|`1`|3|
|||||

n=6을 예로 들자.
|||||||
|:-:|:-:|:-:|:-:|:-:|:-:|
|1|2|3|4|5|6|
|2|*?*|*?*|*?*|*?*|5|
|3|*?*|*?*|*?*|*?*|4|
|||||||

여기서 우린 이제 숫자를 구할 수 있다.
|||||||
|:-:|:-:|:-:|:-:|:-:|:-:|
|1|2|3|`4`|5|6|
|2|`4`|*?*|*?*|*?*|5|
|3|*?*|*?*|*?*|*?*|4|
|||||||

|||||||
|:-:|:-:|:-:|:-:|:-:|:-:|
|`1`|2|3|4|5|6|
|2|4|`1`|*?*|*?*|5|
|3|`1`|*?*|*?*|*?*|4|
|||||||

|||||||
|:-:|:-:|:-:|:-:|:-:|:-:|
|1|2|3|4|5|`6`|
|2|4|1|`6`|*?*|5|
|3|1|*?*|*?*|`6`|4|
|||||||

|||||||
|:-:|:-:|:-:|:-:|:-:|:-:|
|1|2|`3`|4|5|6|
|2|4|1|6|`3`|5|
|3|1|*?*|*?*|6|4|
|||||||

이 다음 부터는 약간 다르다. 이전 배열이 아닌 이전 이전 배열에서 찾아야한다.

|||||||
|:-:|:-:|:-:|:-:|:-:|:-:|
|1|2|3|4|`5`|6|
|2|4|1|6|3|5|
|3|1|`5`|*?*|6|4|
|||||||

|||||||
|:-:|:-:|:-:|:-:|:-:|:-:|
|1|`2`|3|4|5|6|
|2|4|1|6|3|5|
|3|1|5|`2`|6|4|
|||||||

난 이 규칙을 바로 적용해 내번째 알고리즘을 만들었다.

---

### 전체 코드 (Javascript)
```javascript
function createColorMapAlgo(n){
    let out=[[]]; //집합 A 선언
    let half=(n-1)/2;
    let isOdd=n%2===1;
    //집합 A 초기화
    for(let i=0;i<n;i++) out[0][i]=i+1;
    for(let i=1;i<n-1;i++){
        if(i===half){
            let p2=[];
            for(let k=0;k<n;k++) p2[k]=half+1;
            out.push(p2);
            continue;
            
        }
        let p=[i+1];
        for(let k=1;k<n-1;k++) p[k]=[];
        if(isOdd) p[half]=half+1;
        p.push(n-i);
        out.push(p);
    }
    out.push([]);
    for(let i=0;i<n;i++) out[n-1][i]=n-i;
    //집합 A가 초기화됨.
    for(let x=1;x<half;x++){
        for(let y=x;y<n-x;y++){
            let y2=0;
            if(isOdd){
                if(y===half) continue;
                if(y<half){
                    y2=y+2-4*((y-x)%2);
                }else{
                    y2=y+1-4*((y-x-1)%2);
                }
                if(y2>=half)y2++;
            }else{
                y2=y+2-4*((y-x)%2);
            }
            let num=out[Math.max(x-2,0)][y2];
            out[x][y]=num;
            out[n-x-1][n-y-1]=num;
            out[n-y-1][n-x-1]=num;
            out[y][x]=num;
        }
    }
    return out;
}
```
이후 이 코드를 Java로 최적화해 n=10000까지 계산 가능함을 보았다.

```java
import java.io.ByteArrayOutputStream;
import java.io.FileInputStream;
import java.io.FileOutputStream;

public class AMS {
  public static int[][] createColorMap(int n){
    int[][] out=new int[n][n];
    int half=(n-1)/2;
    boolean isOdd=n%2==1;
    for(int i=0;i<n;i++){
      out[0][i]=i+1;
      out[n-1][i]=n-i;
    }
    for(int i=1;i<n-1;i++){
      if(isOdd&&i==half){
        for(int k=0;k<n;k++) out[i][k]=half+1;
        continue;
      }
      out[i][0]=i+1;
      out[i][n-1]=n-i;
    }
    for(int x=1;x<=half;x++) for(int y=x;y<n-x;y++){
      int y2=y+1;
      int mod=(y-x)%2;
      if(isOdd){
        if(y!=half){
          if(y<half){
            y2+=1-4*mod;
          }else{
            y2-=4*((y-x-1)%2);
            if(y2>=half) y2++;
          }
        }
      }else{
        y2+=1-4*mod;
      }
      int num=out[Math.max(x-2,0)][y2];
      out[x][y]=num;
      out[n-x-1][n-y-1]=num;
      out[n-y-1][n-x-1]=num;
      out[y][x]=num;
    }
    System.out.println("calculated");
    return out;
  }
  public static String leftPad(String string, int pad){
    String out="";
    for(int zeroTime=pad-string.length();zeroTime>0;zeroTime--) out+='0';
    return out+string;
  }
  public static String hexColorMap(int[][] colorMap){
    String out="";
    int pad=colorMap.length/16+1;
    for(int[] x:colorMap){
      for(int y:x) out+=leftPad(Integer.toHexString(y),pad);
      out+="\n";
    }
    return out;
  }
  public static String hexColorMapClean(int[][] colorMap){
    String out="";
    int pad=colorMap.length/16+1;
    for(int[] x:colorMap){
      for(int y:x){
        String string=Integer.toHexString(y);
        for(int zeroTime=pad-string.length();zeroTime>0;zeroTime--) out+='0';
        out+=string;
        out+=' ';
      }
      out+="\n";
    }
    return out;
  }
  public static double log2(int x){
    return Math.log(x)/Math.log(2);
  }
  public static void main(String[] args){
    int time=100;
    if(args.length>0){
        time=Integer.parseInt(args[0]);
    }
    try {
        FileOutputStream fin1 = new FileOutputStream("./test1.txt");
        int[][] map = createColorMap(time);
        ByteArrayOutputStream buffer = new ByteArrayOutputStream();
        int pad=((int)(log2(map.length)/4))+1;
        byte[][] sus=new byte[time][pad+1];
        int i=0;
        for (int y : map[0]) {
            String string=Integer.toHexString(y);
            String out="";
            for(int zeroTime=pad-string.length();zeroTime>0;zeroTime--) out+='0';
            sus[i++]=(out+string+" ").getBytes();
        }
        for (int[] x : map) {
            for (int y : x) {
                buffer.write(sus[y-1]);
            }
            buffer.write(new byte[]{0x0a});
        }
        fin1.write(buffer.toByteArray());
        fin1.flush();
        fin1.close();
    } catch (Exception e) {
        e.printStackTrace();
    }

  }
}
```
