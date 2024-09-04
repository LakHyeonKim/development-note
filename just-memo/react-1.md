---
description: 핵사곤을 어떻게 데이터 수량에 맞게 자동으로 배치할까?
---

# 📒 React로 핵사곤 차트 컴포넌트 만들기 1

## 요구사항

* 데이터 수량에 따라 핵사곤이 직사각형 안에 아래 이미지 처럼 배치 되어야 한다.
* 핵사곤이 많아지면 크기가 작아지면서 재배치
* 꼭지점이 위로 올라간 모양으로 그리기

<figure><img src="../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

## 베이스 라이브러리 ([react-hexgrid](https://www.npmjs.com/package/react-hexgrid?activeTab=readme))

* [react-hexgrid](https://www.npmjs.com/package/react-hexgrid?activeTab=readme) svg 기반으로 핵사곤을 쉽게 그려준다.
* 이 라이브러리 특징은 핵사곤 중심 좌표를 Q,R,S 라는 새로운 좌표계로 정의 하고, svg x,y를 q,r,s 좌표계로 변환 하여 사용자는 svg에 시작 포지션만 잡아주면 q,r,s 좌표로 쉽게 그릴 수 있다. (관련 코드 [Hex](https://github.com/Hellenic/react-hexgrid/blob/master/src/models/Hex.ts), [HexUtils](https://github.com/Hellenic/react-hexgrid/blob/master/src/HexUtils.ts), [HexUtils.hexToPixel](https://github.com/Hellenic/react-hexgrid/blob/master/src/HexUtils.ts#L133), [HexUtils.pixelToHex ](https://github.com/Hellenic/react-hexgrid/blob/master/src/HexUtils.ts#L149))
* 핵사곤은 두가지 모양이 있는데 평평한 면이 위로 올라간 Flat-top, 꼭지점이 올라간 Pointy-top 이 존재한다. (중심으로 30도 시계 반대 방향으로 회전 한 차이이다.)
* [핵사곤 그리는 방법](https://www.redblobgames.com/grids/hexagons/) 문서를 참고하면 된다.



## 그려보자

### 임의의 직사각형이 주어 졌을 때 N개의 데이터를 몇개 넣을 수 있을까?



이말은 즉 "임의의 직사각형이 주어 졌을 때 N개의 직사각형을 몇개 넣을 수 있을까" 로 바꿀 수 있다.

핵사곤을 넣어야 하는데 왜? 직사각형일까&#x20;

<figure><img src="../.gitbook/assets/image (11).png" alt=""><figcaption><p>[출처] <a href="https://www.redblobgames.com/grids/hexagons/">https://www.redblobgames.com/grids/hexagons/</a></p></figcaption></figure>

위 그림을 보면 핵사곤을 감싸는 사각형은 직사각형이고 대략적으로 몇개 들어 갈 수 있는지 계산 하기 위해 직사각형으로 보았다.



결론 부터 말하자면 [react-hexgrid](https://www.npmjs.com/package/react-hexgrid?activeTab=readme) 라이브러리는 위 그림과 같이 핵사곤 [size](https://github.com/Hellenic/react-hexgrid/blob/master/src/Layout.tsx#L103)를 넣어주면 크기에 맞게 그려진다. 따라서 N개의 핵사곤이 임의의 직사각형에 적절하게 들어가려면 데이터 수량에 맞는 최적의 [size](https://github.com/Hellenic/react-hexgrid/blob/master/src/Layout.tsx#L103)를 구하여 그려야 한다.



### 핵사곤을 감싸는 개별 직사각형 가로 세로 최적의 길이 찾기



개별 직사각형 가로 세로 길이를 찾으려면 직관적으로 수량의 제곱근을 구해 볼 수 있다.&#x20;

예를 들어 4개를 단순히 큰 사각형에 넣으려면 2 \* 2 즉 가로 2개 세로 2개를 배치하는 것으로 볼 수 있다.

하지만 단순히 제곱근을 구하면 가로 세로 수량이 같을 수 밖에 없다.

직사각형에 넣으려면 세로가 길어지든 가로가 길어지든 길어진 곳에 더욱 많이 배치하고 싶을 것이다.

아래 코드를 보자

```javascript
function calculateOptimalRectSize(
    _width: number,
    _height: number,
    _N: number,
  ) {
    const aspectRatio = _width / _height;
    let cols: number;
    let rows: number;

    if (aspectRatio > 1) {
      cols = Math.ceil(Math.sqrt(_N * aspectRatio));
      rows = Math.ceil(_N / cols);
    } else {
      rows = Math.ceil(Math.sqrt(_N / aspectRatio));
      cols = Math.ceil(_N / rows);
    }

    while (cols * (rows - 1) >= _N) rows--;
    while ((cols - 1) * rows >= _N) cols--;

    const w_prime = _width / cols;
    const h_prime = _height / rows;

    return { w_prime, h_prime, cols, rows };
  }
```

큰 직사각형 가로, 세로 길이를 각각 `_width`, `_height`

이 두 길이의 비율을 `aspectRatio` 이라 부르고 1보다 큰경우는 가로가 긴 직사각형이고, 작으면 세로가 긴 직사각형임으로 `N`에 비율을 곱한 후 제곱근을 구한다 긴쪽이 더 많이 배치 될 수 있도록 하기 위함이다.



이때 올림 처리를 하였기 때문에 실제로 가로 수량, 세로 수량은 조금 클 수 있다.

따라서 가로 수량, 세로 수량을 하나 뺐을 때 충분히 공간이 남으면 하나씩 빼주면서 최적 크기를 구한다.



개별 직사각형의 가로길이 `w_prime`

개별 직사각형의 세로길이 `h_prime`

가로에 들어갈 최적 수량 `cols`

세로에 들어갈 최적 수량 `rows`



이 두 값을 큰 직사각현 가로길이, 세로길이에서 나누어 주면 개별 직사각형의 최적 가로, 세로 길이를 구할 수 있다.



### cols, rows 만 큼 q, r, s 좌표계 만들기

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption><p>[출처] <a href="https://www.redblobgames.com/grids/hexagons/">https://www.redblobgames.com/grids/hexagons/</a></p></figcaption></figure>

[react-hexgrid](https://www.npmjs.com/package/react-hexgrid?activeTab=readme) 라이브러리에서 제공하는 꼭지점이 위로 향했을 때 `q, r, s` 좌표계를 보자&#x20;

핵사곤 `q, r, s` 좌표 합은 항상 0이다.

따라서 `q, r` 만 구하게되면 `s`는 자동으로 구해진다. `s = -q - r`

아래 코드를 보자

```javascript
for (r = 0; r < rows; r++) {
    const offset = Math.floor(r / 2);
    for (let q = -offset; q < cols - offset; q++) {
      if (count === N) break;
      hexes.push(new Hex(q, r, -q - r));
      count++;
    }
  }
```

`r`은 행을 나타내고, `q`는 열을 나타낸다. \
\
`offset`은 직사각형으로 핵사곤을 배치 할때 짝수 행과 홀 수 행이 다르게 배치 되기 때문에 짝 수 홀 수 행에 따른 시작점이 달라 짐으로 `r / 2` 나누고 나머지는 버려준다.



`q`는 이 `offset` 값의 음수 만큼 부터 시작 해야 한다&#x20;

위 그림에서 노란색 핵사곤 중심 부터 아래로 내려가면 바로 알 수 있다.&#x20;

그리고 `offset` 만큼 당겨 그렸으니 `cols - offset` 만큼 그리면 된다.



### pointy top 핵사곤 넓이 공식으로 최적 size 구하기

<figure><img src="../.gitbook/assets/image (11).png" alt=""><figcaption><p>[출처] <a href="https://www.redblobgames.com/grids/hexagons/">https://www.redblobgames.com/grids/hexagons/</a></p></figcaption></figure>



위 그림을 보면 핵사곤을 감싸는 개별 직사각형 w\_prime, h\_prime 길이를 모두 구했고 size는 위 핵사곤 가로, 세로 길이 공식에 대입하면 size를 구할 수 있다.

코드를 보자

```javascript
const size = {
    x: w_prime / spacing / ROOT3 / (width / BASE_SIZE),
    y: h_prime / spacing / 2 / (height / BASE_SIZE),
  };

```

핵사곤 세로 길이 : `2 * size`

핵사곤 가로 길이 : `ROOT3 * size`



간단하게 구하는 공식이지만 추가된 내용이 `spacing`, 큰 직사각형 `width`, `height` 값을 `BASE_SIZE` 로 나눈 수식 까지 추가되어 나누어 진다.&#x20;



`spacing` 경우 [react-hexgrid](https://www.npmjs.com/package/react-hexgrid?activeTab=readme) 라이브러리에서 제공해주는 옵션인데 핵사곤 면이 맞닿는 공간을 띄우는 옵션이다.

이 `spacing` 값은 핵사곤 중심과 중심 거리를 면이 딱 닿아 있을 때 1로 보고 1보다 크면 공간이 떨어지고, 1보다 작으면 침범하게 된다.



위에서 개별 직사각형 `w_prime`, `h_prime` 딱 붙어 있는 즉 `spacing`이 1일 때를 기준으로 계산된 것이고 실제 핵사곤 `size`를 구하기 위해서는 `spacing` 설정 값만큼 역으로 나누어 주어야 한다.



`width`, `height`를 `BASE_SIZE` 를 나누어 주는 이유는 `svg viewbox` 설정 값이 -50 -50 100 100 으로 되어 있는데&#x20;

이는 `svg size`가 `width`, `height`와 같고 이는 좌표계를 `width`, `height` 만큼 확대한 의미와 같다. 따라서 정확한 `viewbox`  기준 `pixel` 값을 구하려면 스케일만큼 나누어 주어야 한다.



여기 까지 구하면 되는데 `size`가 서로 다르면 핵사곤은 찌그러져 이쁜 모양을 그릴 수 없다.

따라서, 아래 코드를 보면

```javascript
let maxSize = Math.max(size.x, size.y);
maxSize *= 0.86 * hexagonMagnification;
maxSize = Math.min(maxSize, hexagonMaxSize);
size.x = maxSize;
size.y = maxSize;
```



두 `size` 중 `max` 값을 취 하고 `hexagonMagnification` `props`를 열어 `size` 스케일을 할 수 있도록 제공 했다.

`maxSize`는 `N` 값이 작을 수록 즉, 데이터 수량이 작을 수록 `size`는 커지게 되는데 이때 `props`로 제공하여 최대 크기를 정할 수 있도록 한다.



### 핵사곤 수량이 많아 질 수록 Origin option 수정



[react-hexgrid](https://www.npmjs.com/package/react-hexgrid?activeTab=readme) 에는 [origin](https://github.com/Hellenic/react-hexgrid/blob/master/src/Layout.tsx#L106) 옵션이 존재 하는데 기본 값은 `viewBox` 기준 가운데 이다&#x20;

여기서 부터 핵사곤들이 그려지고 `N` 수량에 따라 `origin` 위치를 다시 계산하여 수량에 상관 없이 `viewBox` 중앙으로 이동 시켜 야 한다.



아래 코드를 보자

```javascript
const sumHexWidth =
    ROOT3 * x +
    ROOT3 * x * spacing * (cols - 1) +
    (rows > 1 && N >= cols * 2 ? (ROOT3 * x) / 2 : 0);
const sumHexHeight = 2 * y + (3 / 2) * y * spacing * (rows - 1);
const origin = {
    x: -1 * (sumHexWidth / 2) + (ROOT3 * x) / 2,
    y: -1 * (sumHexHeight / 2) + (2 * y) / 2,
  };
```



이미 위에서 핵사곤 가로, 세로 길이 구하는 공식을 알고 `rows`, `cols` 수량 만큼 더한 총 가로 길이 이때 `spacing` 비율도 길이에 포함되어야 한다.

높이도 이와 같은 방법으로 구하면 `sumHexWidth`, `sumHexHeight`를 구할 수 있고 이를 반으로 나누게 되면 중앙 좌표가 그려진다.&#x20;

이때 고려해야할 상황은 핵사곤이 행별로 지그재그로 그려지니 여분 개수가 남게되면 가로 길이에서는 핵사곤 가로 길이 절반 만큼 세로 길이에서는 핵사곤 세로 길이 절반 만큼 중앙에서 우측 하단으로 치우치게 되는데 이를 각각 더해주면 중앙으로 이동된다.&#x20;



### Hex 좌표계, N 에 따른 최적 size, N 에 따른 최적 origin을 구하는 전체 코드

```javascript
export const createCoordinate = (
  width: number, // svg 가로 길이
  height: number, // svg 세로 길이
  N: number, // 데이터 수량
  spacing: number, // hexagon spacing
  hexagonMagnification: number, // hexagon size 스케일
  hexagonMaxSize: number, // hexagon max size
): Coordinate => {
  let count = 0;
  const hexes: Hex[] = [];
  if (!width || !height || !N || !spacing)
    return { hexes, size: { x: 0, y: 0 }, origin: { x: 0, y: 0 } };

  function calculateOptimalRectSize(
    _width: number,
    _height: number,
    _N: number,
  ) {
    const aspectRatio = _width / _height;
    let cols: number;
    let rows: number;

    if (aspectRatio > 1) {
      cols = Math.ceil(Math.sqrt(_N * aspectRatio));
      rows = Math.ceil(_N / cols);
    } else {
      rows = Math.ceil(Math.sqrt(_N / aspectRatio));
      cols = Math.ceil(_N / rows);
    }

    while (cols * (rows - 1) >= _N) rows--;
    while ((cols - 1) * rows >= _N) cols--;

    const w_prime = _width / cols;
    const h_prime = _height / rows;

    return { w_prime, h_prime, cols, rows };
  }

  const { w_prime, h_prime, cols, rows } = calculateOptimalRectSize(
    width,
    height,
    N,
  );
  let r: number;
  for (r = 0; r < rows; r++) {
    const offset = Math.floor(r / 2);
    for (let q = -offset; q < cols - offset; q++) {
      if (count === N) break;
      hexes.push(new Hex(q, r, -q - r));
      count++;
    }
  }
  const size = {
    x: w_prime / spacing / ROOT3 / (width / BASE_SIZE),
    y: h_prime / spacing / 2 / (height / BASE_SIZE),
  };
  let maxSize = Math.max(size.x, size.y);
  maxSize *= 0.86 * hexagonMagnification;
  maxSize = Math.min(maxSize, hexagonMaxSize);
  size.x = maxSize;
  size.y = maxSize;
  const { x, y } = size;
  const sumHexWidth =
    ROOT3 * x +
    ROOT3 * x * spacing * (cols - 1) +
    (rows > 1 && N >= cols * 2 ? (ROOT3 * x) / 2 : 0);
  const sumHexHeight = 2 * y + (3 / 2) * y * spacing * (rows - 1);
  const origin = {
    x: -1 * (sumHexWidth / 2) + (ROOT3 * x) / 2,
    y: -1 * (sumHexHeight / 2) + (2 * y) / 2,
  };
  return { hexes, size, origin };
};

```



## 핵사곤 컴포넌트 전체 소스코드 및 스토리북



핵사곤을 이용한 차트를 많이 사용하는 것 같아 오픈소스를 진행 해보려 한다.

내가 생각한 방식보다 더좋은 방법이 있기를 기대 하며.. ㅎㅎ



{% embed url="https://github.com/nkia-development/hexagon-chart/" %}

{% embed url="https://nkia-development.github.io/hexagon-chart/?path=/docs/example-hexagonchart--docs" %}

## 미리보기



<figure><img src="../.gitbook/assets/화면 기록 2024-09-04 오후 5.46.29.gif" alt=""><figcaption></figcaption></figure>





