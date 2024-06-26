# [REVIEW] 3DGS : 3D Gaussian Splatting for real time radiance field rendering (작성중)
> SIGGRAPH 2023 </br>
> Bernhard Kerbl, Georgios Kopanas, Thomas Leimkühler, George Drettakis</br>
> Inria | Université Côte d'Azur | MPI Informatik

오늘 리뷰하는 논문은 NeRF와 같이 Novel View Synthesis 분야에 엄청난 인기를 끌었던 3D Gaussian Splatting에 대한 논문입니다. NeRF 논문이 발표되고 해당 방법을 개선하는 내용의 논문과 응용한 논문이 엄청나게 쏟아졌었는데, 현재 3DGS를 활용한 논문도 계속해서 발표되고 있어 해당 내용을 정리하고자 진행하게 되었습니다.

3DGS는 NeRF의 task와 동일한, image set과 camera pose값이 주어지면 다양한 시점에 대해서 Rendering을 수행하여 해당 scene을 3D로 표현합니다. 해당 논문에서 제안한 방법의 결과는 고해상도(1080p) Rendering quality 부문 SOTA를 기록한 Mip-NeRF 360 보다 뛰어난 결과를 보였으며, Training time 부문 SOTA를 기록한 InstantNGP 보다 빠르게 학습이 가능합니다.

지금부터 어떻게 이런 결과를 이룰 수 있었는 지에 대해서 이해한 내용을 설명해보도록 하겠습니다.

## Overview
<p align=center>
    <img src="./image/overview.png" width=60% height=60%>
</p>

해당 논문의 전체 흐름은 위의 그림과 같은 순서로 구성되어 있습니다.
4개의 핵심 Block으로 구성되어 있어 어떻게 구성되어 있는지 쉽게 파악할 수 있었습니다.

1. Initialization : 3D Gaussian의 초기값을 설정하는 구간입니다. COLMAP과 같은 SfM(Structure from Motion) 알고리즘을 이용하여 연속된 이미지를 통해 카메라 파라미터와 Point Cloud를 추출하여 초기 3D Gaussian의 값으로 할당해줍니다.
2. Projection : 3D Gaussian을 2D Image plane으로 투영시켜 2D Gaussian으로 변환하는 구간입니다. Image plane으로 투영시켜 Ground Truth 이미지와 비교하여 학습과정에서 모델의 파라미터를 업데이트하기 위한 구간입니다.
3. Differentiable Tile Rasterizer : 해당 블럭의 이름에서 알 수 있듯이, 미분 가능한 Tile들을 Rasterization하여 이미지를 생성하는 구간입니다. 해당 논문의 빠른 Rendering이 가능하도록 한 핵심 아이디어 부분이라고 생각합니다.
4. Adaptive Density Control : 역전파 과정에서 Gradient를 통해 Gaussian의 형태를 업데이트 하는 구간입니다. 해당 논문에서는 Under Reconstruction과 Over Reconstruction의 경우에 대해 Optimization 과정을 거쳐 수행한다고 합니다.

Gradient Flow는 학습 과정에서 역전파로 계산된 Loss에 의해 업데이트하는 과정입니다. </br>

<p align=center>
    <img src="./image/psuedo_code.png" width=60% height=60%>
</p>

`3DGS`의 psuedo code는 위와 같은 흐름을 따라 진행됩니다. 지금부터는 각 단계에 대해 자세하게 알아보도록 하겠습니다.

## 1. Initialization
Initialization block에서는 3D Gaussian을 구성하고 학습에 사용되는 M, S, C, A 파라미터의 초기값을 설정합니다.

<p align=center>
    <img src="./image/initialization.png" width=60% height=60%>
</p>

`M`은 COLMAP과 같은 SfM 알고리즘을 통해 획득한 3D Point Cloud를 초기값으로 설정합니다. Gaussian은 **평균과 분산**으로 구성되게 되는데, 해당 논문에서는 3D Gaussian을 사용하기 때문에 분산이 아닌 공분산으로 구성됩니다. Point Cloud는 3D Gaussian의 초기 평균값으로 사용되며, Point Cloud의 수와 동일한 Gaussian들이 생성됩니다.

`S`는 3D Gaussian의 공분산을 나타냅니다. 3차원에 해당하기 때문에 3x3 크기의 행렬입니다. 

$$ \sum = RSS^{T}R^{T} $$

논문에서는 공분산의 수식을 $\sum$ 기호로 나타냈습니다. Scaling factor로 구성된 vector $S$와 Rotation에 관련된 quarternion을 변환한 $R$로 구성하였고, 독립적인 Opimization을 위해 각 factor들을 따로 저장했다고 합니다.

`S`는 아래와 같은 코드로 값을 할당받게됩니다.

```python
dist2 = torch.clamp_min(distCUDA2(torch.from_numpy(np.asarray(pcd.points)).float().cuda()), 0.0000001)
scales = torch.log(torch.sqrt(dist2))[...,None].repeat(1, 3)
```

`distCUDA2`는 cuda 코드로 작성된 함수로, 평균값을 return해주는 함수입니다. 아마 Point Cloud의 수가 상당히 많아 소요되는 시간을 줄이기 위해 따로 cuda 함수를 작성하여 사용한 것으로 보입니다. 이후, root와 log를 취한 값을 복사하여 3x3 행렬로 만들었습니다.

```python
rots = torch.zeros((fused_point_cloud.shape[0], 4), device="cuda")
rots[:, 0] = 1
```

Matrix `R`은 point마다 크기가 4이고 0으로 초기화된 벡터를 만들어 첫번째 값에만 1로 할당하여 초기값을 세팅하였습니다.

이렇게 공분산 수식을 구성한 이유는 3D Gaussian을 Rendering 할 수 있도록 2D로 변환하는 과정에서 image space에 맞추기 위해서 입니다.(image space가 좌측 상단 (0,0)에서 우측 하단 방향으로 좌표값이 증가하게 되는데, 이를 위해 공분산 행렬이 positive semi definite를 만족하도록(?) 설계했다고 봅니다.)

3D Gaussian을 2D로 Projection하는 수식은 아래와 같습니다.

$$ {\sum}' = JW \sum W^{T}J^{T}$$

$J$는 projective transfomation의 선형 변환을 위한 Jacobian 행렬로 카메라 좌표계를 이미지 좌표계로 변환시켜주는 역할을 합니다.

$W$는 카메라 파라미터를 나타내는 행렬로, 월드 좌표계를 카메라 좌표계로 변환시켜주는 역할을 합니다.

$\sum$은 위에서 구한 월드 좌표계에서의 Covariance를 나타냅니다.

`C`는 3D Gaussian의 색상값을 나타내는데, 해당 논문에서는 Spherical Harmonics(SH)라고 하는 함수로 설계했습니다. SH는 Computer Graphics분야에서 3D 물체가 여러 광원에 영향을 받아 변하는 색상을 실시간으로 계산하기 위해 사용한다고 합니다. 구면 좌표계에서 $\theta$와 $\phi$를 입력 받아 해당 위치의 구면값을 반환하는 함수입니다. 구면좌표계를 라플라스 방정식을 계산하면 아래와 같은 SH 함수($Y_{l}^{m}(\theta, \phi)$)와 확률밀도함수($P_{l}^{\vert m \vert} \cos\theta$)를 얻을 수 있습니다. 수식을 유도하는 과정은 [link](https://elementary-physics.tistory.com/126)에 자세하게 나와있습니다.

$$ Y_{l}^{m}(\theta, \phi) = \sqrt{{(2l+1)(l- \vert m \vert)!}\over{4 \pi (l+ \vert m \vert)!}}P_{l}^{\vert m \vert} \cos\theta e^{im\phi} $$

$$ P_{l}^{\vert m \vert} \cos\theta = (-1)^{m} \frac {(l+ \vert m \vert)!}{(l- \vert m \vert)!}P_{l}^{-\vert m \vert}\cos\theta $$

위의 식에서 $l$은 방위 양자수를 나타냅니다. 오비탈의 모양을 결정하는 양자수로 0 ~ n-1 의 값을 갖습니다. $m$은 자기 양자수를 나타내는데, 음수, 0, 양수 등의 값을 갖고 오비탈의 공간 방향을 나타냅니다. 

<p align=center>
    <img src="./image/spherical_harmonic.png" width=60% height=60%>
</p>

위의 수식의 $\theta$와 $\phi$를 x, y축에 대한 평면으로 표현하여 색을 구성하는 map을 만들어 사용합니다. [Wikipedia](https://en.wikipedia.org/wiki/Table_of_spherical_harmonics)에 따르면 SH 함수의 크기는 Saturation(선명도)을 나타내고 위상은 Hue(밝기)를 나타낸다고 합니다.

[논문](https://3dvar.com/Green2003Spherical.pdf)에 따르면, 최종적으로 `C`는 입력된 $\theta, \phi$에 따라 계산된 각 SH 함수의 결과를 weighted sum하여 결정하게 됩니다. 이때, $l$의 최대값은 고정되어 있기 때문에, 정해진 Y 함수 내에서 가중치 값을 조절하여 색상을 결정합니다.

`A`는 3D Gaussian의 Opacity를 나타내는 값입니다. 초기값은 inverse sigmoid를 사용하여 음수값으로 할당하였는데 특별한 이유는 없는 것 같습니다.

## 2. Projection + 3. Rasterize
다음 과정에서는 Projection block과 Differentiable Tile Rasterizer block을 거쳐 이미지를 생성하고 실제 이미지와 비교하여 Loss를 계산하며 학습을 진행하게 됩니다.

<p align=center>
    <img src="./image/rasterize.png" width=60% height=60%>
</p>

### Sample Training View
카메라 파라미터 `V`와 Ground Truth(GT) 이미지 $\hat{I}$ 를 읽어옵니다. 읽어온 카메라 파라미터 `V`는 위에서 초기화한 `M`, `S`, `C`, `A`와 함께 `Rasterize` 함수의 입력으로 사용되어 이미지 `I`를 생성합니다.

### Rasterize

<p align=center>
    <img src="./image/rasterize2.png" width=60% height=60%>
</p>

실제 함수에서는 이미지 크기를 나타내는 `w`와 `h` 변수들도 입력으로 받게 되어있는데, 초기값이 이미 설정되어 있어 따로 입력값을 전달하지는 않습니다.

해당 논문에서 제공한 알고리즘의 순서를 따라가며 이미지 생성 과정을 설명해보겠습니다.

#### Cull Gaussian
전체 3D Gaussian `p` 중에 입력 카메라 파라미터 `V`에서 관측할 수 없는 3D Guassian을 걸러내는 filter 역할을 하는 함수입니다. 

<p align=center>
    <img src="./image/view_frustum.png" width=60% height=60%>
</p>

주어진 3차원 카메라 정보로 볼 수 있는 평면의 영역인 절두체(Viewing Frustum) 밖에 있는 오브젝트는 렌더링 과정에서 제거(Culling)하게 됩니다. 논문에서는 교챠 영역에서 99% Confidence를 갖는 3D Gaussian만 유지한다고 합니다. 추가적으로, 2D Covariance 연산이 불안정하기 때문에 near plane에 너무 가깝거나 Frustum 밖과 같이 extream한 영역의 Gaussian을 걸러내기 위한 `gurad band`를 사용한다고 합니다.(Frustum Culling에 관한 자세한 내용은 [링크](https://m.blog.naver.com/canny708/221547085908)를 참고하였습니다.)

#### Screen space Gaussians
이 함수는 3D Gaussian을 2D Gaussia으로 변환하여 Image plane에 projection시키는 함수입니다. Scaling Matrix와 Rotation Matrix를 이용하여 3D Covariance를 계산합니다.

$$ \sum = RSS^{T}R^{T} $$

이후, Projective Matrix `J`와 Viewing transformatino `W`와 계산하여 2D로 변환합니다.

$$ {\sum}' = JW \sum W^{T}J^{T}$$

관련 수식은 위에서 설명을 해서 짧게 넘어가겠습니다.

#### Create Tiles
`w, h` 크기의 이미지를 16x16 tile로 쪼개는 함수입니다. 논문에서 제안하는 tile based rasterization을 수행하기 위한 단계로 이해했습니다.

#### Duplicate with Keys
각 2D Gaussian을 겹쳐는 tile 수에 따라 인스턴스화 시킵니다. 각 인스턴스에 view space depth와 tile ID를 결합한 Key를 할당하여 Dictionary 형태로 저장됩니다. Key 정보는 64 bit 크기로 구성되며, 32비트씩 `[ tile ID | depth ]`의 형태로 저장됩니다. 이 과정에서 갹 Gaussian 마다 N개의 인스턴스들이 생기지만, CUDA를 이용한 병렬처리와 다음 단계에서 진행할 정렬 흐름이 더 간단해져서 빠르게 처리가 가능하다고 합니다.

#### Sort by Keys
각 인스턴스에 할당한 Key를 이용하여 Radix Sort를 진행합니다. Radix Sort(기수 정렬)은 낮은 자리수부터 비교하여 정렬하는 알고리즘을 말합니다. 해당 과정에서는 GPU를 이용해 병렬적으로 모든 splat들을 depth에 따라 정렬합니다. 이 과정에서 픽셀마다 point들의 순서는 정해져 있지 않고, blending 과정에서도 초기 정렬을 기반으로 수행된다고 합니다. 저자는 본인들의 $\alpha$-blending 과정에서 일부 구성이 approximate 할 수 있지만, splat이 픽셀 크기에 해당하기 때문에 무시해도 될 정도라고 합니다. 결과적으로 이 방법이 artifacts 줄이고 학습과 렌더링 퍼포먼스에 큰 영향을 끼쳤다고 합니다.

#### Identify Tile Range
같은 tile ID에서 시작과 끝 Gaussian을 식별하여 리스트를 생성하여 효율적으로 관리하기 위한 함수입니다.

#### Get Tile Range
모든 tile에 대한 range `r`을 읽어옵니다.

#### Blend in Order
각 tile 마다 하나의 CUDA thread block으로 실행되며, 주어진 픽셀에 대하여 앞에서 뒤로 순회하며 색상과 투명도($\alpha$)값을 축적하여 값을 결정합니다. 이때, 데이터 loading, sharing, processing 등을 병렬로 처리하여 이득(속도 측면에서의 이득이라고 생각합니다.)을 최대화한다고 합니다. 픽셀이 목표 채도 $\alpha$에 도달하면, 연관된 thread의 동작을 중지시키고 모든 픽셀이 만족한다면 전체 tile에 대한 처리를 중지합니다.

해당 논문에서는 Gaussian을 사용한 `빠르고 미분가능한 Rasterizer` 구현을 목표로 했습니다. [previous work](https://arxiv.org/abs/2004.07484)의 문제점인 gradient를 update 할 수 있는 splat의 수가 정해져 있는 문제를 해결하기 위해서 전체 이미지에 대하여 primitive를 pre-sort하는 방법을 사용했다고 합니다. 이렇게 하면, 적은 양의 메모리만으로도 임의 수의 Gaussian에 대하여 효율적으로 역전파를 수행할 수 있고 픽셀마다 constant overhead가 소요된다고 합니다. 이렇게 해서 미분가능하고 2D로 projection이 가능하기 때문에 anisotropic splat이 가능하게 됩니다.

anisotropic은 비등방성이라는 뜻으로, "속성이 방향에 따라 변하는" 이라는 뜻으로 볼 수 있습니다. 논문에서는 정확한 scene 표현을 위해 방향에 따라 물체의 색상이 바뀌기 SH 함수가 이와 관련이 있다고 볼 수 있습니다.

Backward pass 단계에서는 forward pass 에서 수행했던 픽셀당 blended point의 전체 sequence를 복구해야한다고 합니다. 이를 해결하기 위해서는 [기존 방법](https://arxiv.org/abs/2109.02369)에서 사용한 global memory에 point를 저장하는 방법도 있지만, dynamic memory management overhead가 발생할 수 있다고 합니다. 이를 피하기 위해 forward pass의 tile range와 sorted array of Gaussia을 재사용하여 list를 뒤에서 앞으로 탐색하는 방법을 사용한다고 합니다.

### Loss
생성된 이미지와 GT 이미지를 비교하여 차이를 계산합니다. Loss 함수에는 L1과 D-SSIM을 사용하여 설계했습니다. D-SSIM은 SSIM을 기반으로 한 함수로, 밝기/대비/구조 등을 기반으로 유사도를 계산하는 함수입니다.

### Adam optimizer
Loss 계산 후에는 Adam optimizer를 사용하여 `M`, `S`, `C`, `A`를 업데이트 합니다. Opimization Detail은 [논문](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/3d_gaussian_splatting_low.pdf)의 7.1절을 확인해보시면 나와있습니다.

## 4. Adaptive Control Gaussian
마지막으로 Gaussian을 주어진 scene에 맞게 adaptive control하는 block 입니다.

<p align=center>
    <img src="./image/adaptive_control_gaussian.png" width=60% height=60%>
</p>

이 과정은 학습 과정에서 매 번 수행되는 게 아니라 100번마다 수행되도록 설계했다고 합니다. 이 과정에서 transparent한 Gaussian($\alpha$가 threshold보다 낮은)을 제거하게 됩니다. 이후, empty area를 채우도록 adaptive control을 수행하게 되는데, geomotric feature가 없는 경우(under-reconstruction)과 Gaussian이 커버한 영역이 광범위한 경우(over-reconstruction)을 고려하며 진행됩니다.

논문에서는 이 과정을 Densification이라고 정의했습니다. threshold $\tau_{pos}$가 0.0002를 넘어선 view-space position gradients의 average magnitude를 갖는 Gaussian에 대해 adaptive control을 수행합니다.

#### Under-reconstruction
작은 Gaussian, 해당 포인트에서 Gaussian의 크기를 나타내는 Covariance가 작은 Gaussian들은 같은 크기로 복사(Clone)해서 부족한 geometry를 cover한다고 합니다. 이때, 복사한 Gaussian의 위치는 positional gradient 방향이 됩니다.

#### Over-reconstruction
너무 크기가 큰 Gaussian에 대해서는 작은 크기의 Gaussian으로 쪼개서(Split) 사용합니다. 쪼개진 Gaussian의 크기는 실험을 통해 결정된 scale factor $\phi$(=1.6)로 조절된다고 합니다. 쪼개진 Gaussian은 원래 Gaussian의 확률밀도함수(Possiblity density function)에 따라 위치하게 됩니다.

## Result and Evaluation

<p align=center>
    <img src="./image/result.png">
</p>

정량적 평가에 대한 결과는 위의 사진과 같이 거의 모든 경우에 3D Gaussian이 우수한 것을 알 수 있습니다. 특히 학습속도가 빠른 InstantNGP 보다도 빠른 것을 볼 수 있습니다. 다른 방법들 보다 Rendering 속도가 굉장히 빠른 것이 큰 장점인 것 같습니다. 반대로, 다른 방법들에 비해 메모리 사용량이 굉장히 큰 것을 볼 수 있습니다.

<p align=center>
    <img src="./image/result2.png" width=60% height=60%>
</p>

학습 결과에 대한 사진을 논문에서 제공했지만, 개인적인 생각으론 [Project page](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/)에서 보시는 것이 직관적으로 더 와닿는 것 같습니다.

## Limitation
저자들은 scene이 잘 관찰되지 않은 경우(=sparse한 scene)에 대해 artifact가 생긴다고 합니다. 이러한 증상은 다른 방법들에서도 나타난다고 합니다. Anisotropic Gaussian이 큰 이점을 주긴 했지만, 논문에서 표현하길 길쭉하거나 얼룩진 Gaussian을 생성할 수 있다고 합니다.

Optimization 과정에서도 큰 Gaussian을 만들 때 artifact가 발생하고, 특히 view-dependent appearance에서 나타나는 경향이 있다고 합니다. 이러한 현상은 rasterize 단계에서 gaurd band에 의해 Gaussian이 제거하는 과정이 원인이 될 수 있다고 합니다. 또한, Visiblity algorithm에서 갑자기 depth/blending order를 뒤바꾸는게 원인이 될 수 있다고 합니다.

## Comment
이번에는 3D Gaussian Splatting에 대한 논문을 리뷰해보았습니다. 처음 들어보는 개념이 많이 있었고 선행되는 지식이 많이 필요했던 논문이라 몇 번을 읽으며 이해해보려 노력했습니다. 결과적으로 Rendering 속도가 굉장히 빠른 것이 신기하고 이미지 퀄리티고 나쁘지 않아서 신기했는데, 다른 방법들에 비해 메모리가 많이 사용된다는 점과 이 외에도 문제점이 많다는 점이 아쉬운 논문이였습니다.
하지만, 최근 지속적으로 Gaussian splatting, Surfel 등과 같이 새로운 표현 방법을 도입해서 scene을 rendering 하는 방법들이 발표되고, 결과도 계속해서 좋아지고 있습니다.(너무 빨라서 따라가기 힘들어요..ㅜ)
하나의 방법에서 발생하는 문제점들을 극복하는 방향으로 계속 논문이 발표되고 있어서 차근차근 하나씩 리뷰해보도록 하겠습니다.(리뷰 속도도 올려보겠습니다..!)