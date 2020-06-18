+++
title = "Machine learning"
description = ""
tags = [
    "Deep learning",
    "Machine learning",
]
date = "2018-06-21"
categories = [
    "AI",
]
highlight = "true"
+++
머신러닝/딥러닝의 개념을 이해하기 위해 만든 자료이다.
홍콩과기대 김성훈 교수님의 [모두를 위한 머신러닝과 딥러닝의 강의](http://hunkim.github.io/ml/)를 바탕으로 만들어졌고,
Backpropagation 부분은 Andrew Ng교수님 강의 자료 등 아래 링크를 참조하였다.
1. [Machine Learning — Andrew Ng, Stanford University](https://www.youtube.com/watch?v=0twSSFZN9Mc&t=176s&index=51&list=PLLssT5z_DsK-h9vYZkQkYNWcItqhlRJLN)
2. [CS231n Winter 2016: Lecture 4: Backpropagation, Neural Networks 1](https://www.youtube.com/watch?v=i94OvYb6noo)
3. https://mattmazur.com/2015/03/17/a-step-by-step-backpropagation-example/

# Linear Regression

공부를 많이 할 수록 성적이 올라간다.

![gradient](/images/simple-linear.png)

## Hypothesis

Hypothesis는 아래와 같이 1차 방정식이다.
```
H(x) = Wx + b
```

![Linear Regression](/images/linear-regression.png)

즉, 위의 그래프 처럼, Data points에서 가장 cost가 적은 Linear한 함수를 만들기 위해 W와 b를 찾아야 한다.
방법은 cost function과 Gradient descent algorithm을 이용한다.

## Cost function

임의의 W와 b를 대입한 Hypothesis의 linear한 함수(H(x(i)))에
각각의 Data point(y(i))의 거리(차이)를 제곱하여 평균을 구한 값이다.
즉, 아래 식이다.

![Cost Function](/images/cost-function.png)

## Gradient descent algorithm

Cost function에서 임의의 W와 b를 넣어서 cost를 계산하였다.
이것을 반복하여 가장 cost가 적은 W와 b를 찾는것이 이 알고리즘의 목적임.
- Cost function에서 제곱을 했으므로 Data points와 임의의 W와 b를 대입한 H(x)의 값이, 차이가 크면 클수록 기하급수적으로 안좋은 결과가 나온다.
그래서 결과적으로 아래와 같은 2차함수 그래프가 될것이다.

![gradient](/images/gradient.png)

- 그럼 어떻게 임의의 값에서 시작해서 가장 cost가 적은 W와 b를 찾을 수 있을까? 그건 위의 그래프에서 미분을 하여 기울기를 구하고 기울기가 0이 될때 까지 반복하면 된다.

1) 일단 수식을 간단하게 하기 위하여
	- b를 제외하고 W만 고려
	- 나중에 미분을 적용할 때 수식을 간단하게 하기 위해서 1/m을 1/2m으로 변경한다(2를 곱한다고 크게 달라질게 없으므로).
    ![gradient_1](/images/gradient_1.png)
2) W에서 cost(W)를 미분한것을 빼는 식이다. 이유는 이차방정식 그래프에서 오른쪽에 있다면 미분결과가 (+)이므로 x축에서 (-)해야 하고 왼쪽에 있다면 그 반대이기 때문,
그래서 기울기가 크면 클수록 더 많은 값을 빼거나 더해서 최종 미분값이 0에 가까워질때까지 descent한다.
알파는 learning rate이라는 상수라는데 미분한 값으로만 이동하면 너무 느리니까 이것을 곱해서 좀 더 빨리 이동하도록 하는것 같다.
다음 W값을 정하는 방법은 아래처럼 **기존 W값에서 W에 Cost를 미분한 값에 알파값을 곱한 후 (-)한 값**
    ![gradient_2](/images/gradient_2.png)

## Convex function

아래와 같은 그래프에서는 경사도를 잘못 타면 최저 cost값을 계산못 할 수도 있다.

![convex-function_1](/images/convex-function_1.png)

그래서 cost function 의 모양이 아래처럼 밥그릇 모양의 convex function인지 먼저 확인해야 한다.

![convex-function_2](/images/convex-function_2.png)

# Multi-variable linear regression

아래처럼 여러개의 입력값(x1, x2, x3)이 있을때 Y를 어떻게 예측할 수 있을까?

![Multi-variable_1](/images/multi-variable-table.png)

## Hypothesis

- 1개가 있을땐 W를 하나만 받으면 되지만, 복수개일 때는 w를 여러개 받으면 된다.

![Hypothesis](/images/multi-variable-hypothesis.png)

- 근데 식으로 나타낼때 입력값이 너무 많으면 굉장히 길어지므로 matrix(대문자)를 이용하여 간략하게 표현한다. (b는 일단 여기서도 생략함)

![hypothesis-using-matrix_1](/images/hypothesis-using-matrix.png)

- 입력 instance(DB에서 말하면 tuple)가 여러개일 때도 matrix로 하면 된다.

![hypothesis-using-matrix_2](/images/hypthesis-matrix-2.png)

위에서 결과 [5, 1]은 앞에 두개 [5, 3]에서 `5`, [3, 1]에서 `1`이다.
instance가 n개 이면 위 예제에서 5대신에 `n`을 넣으면 된다.

## Cost function 도 간단히 여러개 받아서 y(i)를 빼고 제곱하면 된다.

![cost function](/images/multi-variable-cost-function.png)

# Classification (Binary 1 or 0)

- 메일이 spam or ham.
- facebook에서 show or hide.
- credit card의 사용 패턴을 보고 일반적인 사용인지 아닌지
## Logistic Hypothesis

- 기존의 Linear Regression의 hypothesis는 0과 1사이의 값이 아니라 문제가 되고, 아주 큰값이 있을 경우 Linear 함수의 기울기가 크게 바뀌므로 문제가 될 수 있다.
- 그래서 별도의 Hypothesis 즉 0과 1사이로 압축을 시켜주는 식이 필요함
- 식

![sigmoid-function](/images/sigmoid-function.png)

- 그래프

![sigmoid-graph](/images/sigmoid-graph.png)

logistic function 또는 sigmoid function 이라고 함.
즉, z가 커지면 g(z)는 1에 가까이 가고,
z가 0이하로 작아지면 g(z)는 0에 가까이 가게 된다. (W와 b로 0 이하값으로 조정 가능)

## Decision boundary

(모두를 위한 머신런닝/딥 러닝 강의"에는 없어서 구글링 해서 추가함)

Logistic hypothesis 그래프를 보면 결국엔 z값이 0보다 크면 1이고 아니면 0이다.
그래서 만약에 x1, x2 두개의 input이 있다면, H(x1, x2) = g(x1w1 + x2w2 + b)일 것이고,
뒤에서 설명할 cost function에서 찾은 값이 (-3, 1, 1) 이라면 아래 그래프처럼 Decision Boundary(핑크색 줄)가 결졍되고 이 기준으로 1과 0이 구분 될 것이다.

![Decision-boundary](/images/logistic-decision-boundary.png)

## Cost function

- 기존의 Linear Regression의 Cost function을 그대로 사용할 수 없는 이유
	- Logistic Regression에서는 sigmoid-function은 0과 1 사이의 대충 S자 모양의 그래프로 이걸 제곱을 하면 그 구부러지는 정도가 더욱 심화 될것이고 그러면 경사타고 내려가다가 최저점에 도달하기 전에 기울기가 0이 되는 구간이 생길 수 있어서 Linear Regression의 Cost function을 사용할 수 없다. 아래 참조

![logistic-cost-function-0](/images/logistic-cost-function-0-1.png)

- 그래서 1일때와 0일때를 나눠서 아래 식을 이용함.
	- 즉 상용로그, log(1/H(x)) = -log(H(x))는 H(x)가 커지면 커질 수록 0에 가까워 진다.
	반면은 반대쥐.

![logistic-cost-function-1](/images/logistic-cost-function-1.png)

- 근데 두개를 나눠서 프로그램에 적용하려면 if 를 써야 하니까 아래처럼 하나로 만듦
	- **C(H(x),y) = - ylog(H(x)) - (1-y)log(1-H(x))**
	- 즉, y가 1이면 앞에것이 적용될것이고, 0이면 뒤에것이 적용되겠지
- 그래프로 나타내면 (출처:http://copycode.tistory.com/162) 아래처럼 돼서, Linear regression처럼 미분해서 경사타고 내려가기로 최저 cost를 구할 수 있음

![logistic-cost-function-graph](/images/logistic-cost-function-graph.jpeg)

- 이제 미분해서 최저 cost를 구하면 된다. (알파는 다음값으로 얼마나 이동할지에 대한 상수)
	- 즉, 다음 Weight값은 현재 W에서 W기준으로 cost를 미분해서 알파를 곱한 값을 뺀 값으로 결정된다.

![logistic-cost-function-3](/images/logistic-cost-function-3.png)

# Multinomial classification (Softmax classification)

말 그대로 위에서 설명한 classification에 input이 여러개인것

## Multinomial 개념

- 아래처럼 3개가 있다면, 각각의 input별로 구분해서 Decision boundary를 구하면 됨.

![multinomial-classification](/images/multinomial-classification.png)

- 계산은 3개를 행렬 곱으로 하면 간단해 지네
헷갈리지마, x가 input값이고, w가 Y와 가장 유사한 가설 Hypothesis (Y에 모자를 씌운) 을 찾기 위한 변수임, 그래서 세로의 x값이 여러개이면 뒤로 쭉 붙겠지
결국엔 "A인지 아닌지, B인지 아닌지, C인지 아닌지"에 대한 각각의 w값을 찾는거쥐.

![multinomial-classification-2](/images/multinomial-classification-2.png)

- 결국은 아래처럼 가설 Hypothesis (Y에 모자를 씌운)를 계산하는것임.

![multinomial-classification-3](/images/multinomial-classification-3.png)

Sigmoid함수를 어떻게 적용해야 할까는 다음 참조

## Cost function

이제 Cost function을 이용해서 각 Y와 가장 유사한 가설 Hypothesis (Y에 모자를 씌운) 을 찾아야 한다.
다른말로 해서, 각 Y에 대한 Decision boundary를 찾아야 한다.

### Softmax

- 아래는 아무것도 적용 안하고 곱하기만 한 결과이다.

![multinomial-cost-function-1](/images/multinomial-cost-function-1.png)

- 이것을 0~1 사이의 확률로(모두 합하면 1이 되는) 표시하려면,

![multinomial-cost-function-2](/images/multinomial-cost-function-2.png)

- 이것을 0~1 사이의 확률로(모두 합하면 1이 되는) 표시하려면, 아래의 `Softmax`함수를 이용한다.

![multinomial-softmax-1](/images/multinomial-softmax-1.png)

- one-hot encoding기법을 활용해서 확률이 가장 큰 것을 1.0로 해서 하나를 고를 수 있다.(이건 쉽쥐)

![multinomial-softmax-2](/images/multinomial-softmax-2.png)

### Cross entropy cost function

- 이제는 cost-function을 이용해서 계산한 값(S(y) = Y에 모자)과 실제 값(L = Y) 비교해서 얼마나 유사한가`D(S,L)`를 계산한다.(반복해서 계산하여 가장 유사한 W 백터를 찾아야 위해)
- 얼마나 유사한가를 계산하기 위해서 여기선 `CROSS-ENTROPY`를 이용한다.

![multinomial-cross-entropy-1](/images/multinomial-cross-entropy-1.png)

- 아래와 같이 계산된다.

![multinomial-cross-entropy-2](/images/multinomial-cross-entropy-2.png)

- 즉, 실제 값(Y)에 -log(0~1사이의 예측한값) 이다.
- Y값은 배열로 해당한지 안하는지를 0과 1로 표현될것이다.
    - 예를 들어: [0,1,0,0]
- 그리고 예측한 값은 Softmax를 이용하여 0~1 사이의 실수인 확률(합한 값이 1인)로 표현될 것이다.
    - 예를 들어: [0.2, 0.6, 0.1, 0.1]
- 예측한 값은 앞에 -log가 있으므로 위의 그래프처럼 1에 가까워지면 0에 가까워 질것이고, 0에 가까워 지면 무한대로 값이 커질 것이다.
- 그래서 [0,1,0,0] 과 -log([0.2, 0,6, 0.1, 0.1])를 각 항목별로 곱하기를 하면 1에 가까운 값이 가장 작아 질 것이다.
- 아래는 교수님이 간단히 항목 2개로 증명한 것이다.

![multinomial-cross-entropy-3](/images/multinomial-cross-entropy-3.png)

- 그럼 이전에 설명했던 Logistic regression의 cost function과 지금 설명한 cross-entropy는 증명해 보면 결국엔 같은 것이다.
> http://mazdah.tistory.com/791 참조

![multinomial-cross-entropy-4](/images/multinomial-cross-entropy-4.png)

### W 백터 구하기

- 이젠 이 cost funtion으로 구한 값(즉, 실제값고 추측값의 차이)이 가장 적은 W 백터를 구해야 한다.
- 일단 입력값의 instance(db에서는 tuple)가 여러개일 경우 모두 합해서 1/n을 해서 cost를 구해야 한다.

![multinomial-cross-entropy-5](/images/multinomial-cross-entropy-5.png)

- 그리고 Gradient descent 알고리즘으로 최저 cost값이 되는 W와 b를 찾는다. (알파값으로 다음값을 선택하여.... 알파값 선택하는 방법이 꽤 중요할듯 하다.)

![multinomial-cross-entropy-6](/images/multinomial-cross-entropy-6.png)

# ML Tips

## Learning rate 정하기

Gradient descent algorithm에서 다음값을 선택하는 상수인 알파값이 learning rate이다. 이것을 어떻게 정하느냐가 매우 중요함
- Overshooting: learning rate을 큰값으로 줘서 최저값으로 내려가는게 아니라 바깥으로 튕겨져 나가버리는것.
- 너무 작은값으로 줘도 성능이 떨어진다.
- learning rate을 정하는데 정해진 방법은 없다. 처음엔 0.01을 줘보고 고쳐 나가야 한다네.

## Data(X) preprocessing for gradient descent

- 입력 데이터(여기선 Feature data라고 하네)를 선처리 해서 연산이 수월하게 할 수 있다.
- 여러번 얘기 했듯이, gradient descent 알고리즘으로 W값을 찾는다.
만약 입력 데이터가 x1, x2 두가지가 있어서, 찾고자 하는 W도 w1, w2 두가지라면 알파값으로 찾아가므로 아래 그림처럼 등고선 형태가 될것이다.(가운데가 최저 cost)

![data-preprocessing-1.png](/images/data-preprocessing-1.png)

- 근데 아래처럼 하나의 attribute의 값이 굉장히 크고 기복이 심하면 등고선 형태가 납작해져서 알파값으로 잘못 이동하면 순차적으로 가운데를 향하는게 아니라 삐져나올 수 있다. 즉, 위에서 말한 Overshooting될 수 있다.

![data-preprocessing-2.png](/images/data-preprocessing-2.png)

- 입력 데이터를 조금 수정할 필요가 있다. 여기선 Normalize한다 라고 함.

### Normalize

- Normalize방법은 아래 그림처럼 두가지가 있다.

![data-preprocessing-3.png](/images/data-preprocessing-3.png)

1. zero-centered data : data의 중심을 그래프의 중심(?)으로 이동시키는 방법
2. nomalized data : data가 어느 영역안에 포함된 것, 가장 많이 사용함
	- Standardization: data를 어느 영역안으로 포함시키는 방법
	- 즉, ((입력데이터 - 계산한 평균값) / 계산한 분산의 값) 하면 된다.
	- 아래 처럼 Python으로 쉽게 적용 가능함

![data-preprocessing-4.png](/images/data-preprocessing-4.png)

## Overfitting

- 학습 데이터에 너무 지나치게 잘 맞게 Decision Boundary정해져 있어서, 일반적인 데이터를 적용하면 적확도가 떨어지는 모델
- 아래 오른쪽이 overfitting 모델임

![overfitting.png](/images/overfitting.png)

- overfitting을 줄일 수 있는 방법
	- More training data
	- reduce the number of teatures (중복되는 data를 줄인다던지)
	- **Regularizatoin**

### Regularization

- 일반화 하는것임, 즉 너무 큰 데이터를 수정해서 일반화 시킴.
이렇게 하면 위에서 overfitting 모델에서 구부러진 Decision Boundary를 좀 펼 수 있다.
- 방법은 간단하다. 큰 값의 x(i)값이 있을 때, 그 값이 w(i)를 정하는데 영향을 들 미치게 하는것이다.
아래 식을 보면 뒤에 W(가중치)들을 제곱한 걸 모두 합한 값을 더해주고 있다.
즉, 큰 값의 x(i)기준으로 w(i)를 정하게 되면 w(i)값에 제곱 값이 + 되니 cost가 높아져서(위의 그림에선 loss) 최종 w(i)가 될 확률이 낮아 진다.

![c](/images/regularization.png)

regularization strength를 상수(위 그림에서 람다)로 줘서 cost값을 정할때 어느정도 가중치를 줄지 정할 수 있다.

# Training/Testing Data sets

## Training, validation and test sets

- ML 모델을 data를 가지고 학습을 시켰는데, 이게 정말 유효한지, 정말 잘 동작하는지 확인이 필요하다.
그래서 아래처럼 학습 data를 2 or 3가로 나눈다.
	- Training set: 실제 ML 모델을 학습시키기 위한 data set
	- Validation set: 다음 cost function의 w값을 정하기 위한 알파값과, regularization strength를 위한 람다값을 정하기 위한 data set
	- Testing set: 학습 모델이 잘 동작하는지 확인 하기 위한 data set

![training-set.png](/images/training-set.png)

## Online learning

- 학습 data를 한꺼번에 넣지 않고 나눠서 학습 시키는것, 병렬로 한꺼번에 하는것이 아니라, 이전에 학습한것이 모델에 적용된 이후에 다음것을 학습시킴.

![online-learning.png](/images/online-learning.png)

## 실제 dataset

- 아래는 실제 MINIST에서 사용하는 dataset임

![dataset-example-minist.png](/images/dataset-example-minist.png)

# Neural Network

- 인간의 neuron처럼 아래와 같이 동작시킨다.
	- 어떤 neuron에 input들이 있음.
	- 각 input은 신호의 세기(W)가 다름
	- 이 input들의 신호의 세기의 합에 b를 더함
	- 더한 값이 어느 threshold값을 넘어가게 되면 다음으로 전달 아니면 전달 안함

![neuron-1.png](/images/neuron-1.png)

- 그래서 아래처럼 Activation function을 만듦

![activation-function.png](/images/activation-function.png)

## NN 난제들

### 난제 1. XOR 같은 형태의 Training data와 Target data는 W(Weight)와 b(bias)를 찾기가 어려움

아래처럼 AND 와 OR 은 Linear하게 구분이 되지만, XOR은 하나의 선으로 구분이 불가능 하다.

![xor-problem-1.png](/images/xor-problem-1.png)

처음엔, 왜 XOR을 적용해야 하지? 라는 의문이 들었다. (https://blog.thoughtram.io/machine-learning/2016/11/02/understanding-XOR-with-keras-and-tensorlow.html)
이유는 간단하다. Toy problem (현실 문제를 간단하게 만든것)일 뿐이다.
예를 들어서 아래 그래프 처럼 linearly separable 이 불가능 한 경우 단일 perceptron 으로는 Decision boundary를 찾을 수 없기 때문에
MLP(Multilayer perceptrons)을 이용해야 하는데
문제는 각 Layer의 bias값과 각 Node의 Weight값을 계산하기가 너무 복잡함.

![xor-problem-1.png](/images/xor-problem-9-1.png)

![xor-problem-1.png](/images/xor-problem-10-1.png)

#### XOR을 MLP(Multilayer perceptrons)로 Decision boundary 찾기

일단, MLP(Multilayer perceptrons)을 이용하여 XOR의 Decision boundary 찾을 수 있는지 살펴보자.
- data는 아래와 같다.
```
training_data = np.array([[0,0],[0,1],[1,0],[1,1]], "float32")
target_data = np.array([[0],[1],[1],[0]], "float32")
```
- 한개로는 불가능 하지만, 아래처럼 여러개로 구성된 MLP(Multilayer perceptrons)로는 할 수 있다.
(W와 b값을 구하는 방법(backpropagation)은 뒤에서 설명함)

![xor-problem-2.png](/images/xor-problem-2.png)

- 정말 되는지 계산해보자
- [0,0]일 때 ([0,1],[1,0] 은 똑같은 방법으로 풀면 된다.)

![xor-problem-4.png](/images/xor-problem-4.png)

- [1,1]일 때

![xor-problem-5.png](/images/xor-problem-5.png)

- 이걸 뭉쳐서 그려보면 아래처럼 될것이다.

![xor-problem-6.png](/images/xor-problem-6.png)

- 이걸 하나로 하려면 행렬 곱을 하면 된다. (b 는 생략함)

![xor-problem-7.png](/images/xor-problem-7.png)

- 이걸 수식으로 나타내면 아래와 같다.

![xor-problem-8.png](/images/xor-problem-8.png)

- 이걸 tensorflow python으로 구현하면 두줄로 표현됨.

```
K = tf.sigmoid(tf.matmul(X, W1) + b1)
hypothesis = tf.sigmoid(tf.matmul(K, W2) + b2)
```

#### Multilayer perceptrons에서 W와 b를 찾는 방법

- 1982 by Paul과 1986 by Hinton이 아래의 back propagation으로 해결함

![backpropagation.png](/images/backpropagation.png)

- 즉, 최종 결과가(dog)가 틀렸다면 각 perceptron의 W와 b가 조절해야 하는데, 이걸 입력부터 찾아 가는것이 아니라 뒤에서 부터 미분을 해서 앞으로 진행하면서 W와 b값을 조절해 나가는 방식 -> Back propagation
- 상세한 설명은 저기~ 아래에 있음

### 난제 2. Backpropagation에서 Activation function 문제

- Many layers에서는 뒤에서 부터 W와 b값을 조절해 가는 Backpropagation기법을 사용할 때,
즉, 최종 결과인 error를 뒤로 보낼 때 sigmoid함수를 통하면서 값이 점점 압축하다 보니 앞쪽의 perceptron에 도달하기 전에 사라져 버리는 문제

![backpropagation-1.png](/images/backpropagation-1.png)

#### 해결

- 뒤에서 설명하겠지만, Activation function으로 sigmoind 함수가 아니라 ReLU나 다른 함수를 사용

# Backpropagation

아래 링크를 참조하였다.
1. [CS231n Winter 2016: Lecture 4: Backpropagation, Neural Networks 1](https://www.youtube.com/watch?v=i94OvYb6noo)
2. https://mattmazur.com/2015/03/17/a-step-by-step-backpropagation-example/
3. [Machine Learning — Andrew Ng, Stanford University](https://www.youtube.com/watch?v=0twSSFZN9Mc&t=176s&index=51&list=PLLssT5z_DsK-h9vYZkQkYNWcItqhlRJLN)

## 개념 이해

### 기본 예제

`f(x,y,z)=(x+y)z` 가 있을때 각 입력값이 최종 output f 에 미치는 영향, 즉, 미분값을 어떻게 구할 수 있을까?
문제는 각 x,y,z로 직접 output f를 미분할 수 없으므로 아래와 같이 Backpropagation으로 계산하여야 한다.
1. forward pass로 계산해 나간다. (녹색 값)
2. 각 게이트의 입력값들이 output에 미치는 영향을 계산하기 위해서 최종 output에서 backward pass로 미분해 나가며 계산한다. (붉은색 값)
	- (+) 게이트는 forward pass 의 입력값들이 무엇이든 상관없이 이들에게 backward pass 입력값을 똑같이 넘겨준다.
	- (*) 게이트는 편미분해서 넘겨준다.

![bp-basic-01.png](/images/bp-basic-01.png)

### 실제 사용 예제

그럼 실제와 같이, sigmoid Activation function에 x1, x2 입력값과 w1, w2 가중치를 Activation function을 통과하는 예를 보자.
- 여기서 중요한 것은 x로 최종값 L을 직접 미분할 수 없다.
 (x, y는 각각 x0*w0, x1*w1가 될것이다.)

![bp-basic-02.png](/images/bp-basic-02.png)

- 이것을 위에서 설명했듯이 forward pass / backward pass 로 계산한다. 

![bp-basic-03.png](/images/bp-basic-03.png)

- 근데 sigmoid function을 미분하여 아래와 같이 노드를 간략화 시킬 수 있다.

![bp-basic-04.png](/images/bp-basic-04.png)

## Backpropagation in Neural Network

- 참고자료
    - https://mattmazur.com/2015/03/17/a-step-by-step-backpropagation-example/
- 예제 모델
![backpropagation-cost-function-1.png](/images/backpropagation-1-1.png)

### 1. Forward Pass 로 최종 cost(Error)값를 구한다.

- W(처음엔 초기값을 가지고)와 b를 가지고 Input layer부터 Output Layer까지 `Forward Pass`로 `net`과 `out`값을 계산하여 최종 output Layer의 각 노드의 `out`을 구한다.
	> `net`과 `out`사이에 activation (sigmoid)함수가 있다.

- 그리고 Output layer의 out값으로 [cost function](#cost-function)을 이용하여 cost(Error)를 계산한다.

#### Cost function <a name="cost-function"></a>

- Andrew Ng교수님이 언급하셨듯이, Linear regression의 cost function을 사용해도 비슷한 결과가 나오긴 한다.
하지만 Gradient descent 에서 local minimum 문제가 발생할 수 있으므로 Logistic regression의 cost function을 사용할 것이다.
참고 사이트에는 cost function으로 linear regression의 cost function을 사용해서 수정하였다.
- 즉, 아래와 같다. (상세한 설명은 [Cost function for neural network](#nn-cost-function) 참조)
Etotal: cost(Error), k: output layer의 out들, y: 실제 값, h세타(x): 계산된 out값

![backward-pass-3.png](/images/backward-pass-2-2.png)

### 2. Output layer에서 Backward Pass 로 W값을 조정

- w값을 조정하기 위해서는 현재 각 w값이 cost(Error)에 미치는 영향을 알아야 한다. 
즉, 각 w 기준으로 cost를 미분하여 Gradient descent로 cost가 제일 적은 w를 찾는 과정을 반복하여야 한다.
- 예제 모델에서 Output layer의 `o1`을 예를 들어 보자.

![backward-pass-2.png](/images/backward-pass-2-1.png)

- 즉, W5가 cost에 미치는 영향은 아래 [chin rule](#chain-rule)로 구할 수 있다. (cost(W) 가 Etotal로 치환됐다.)

![backward-pass-4.png](/images/backward-pass-2-1-1-1.png)

- w5 를 수정한다.

![backpropagation-gradient-descent-1.png](/images/backpropagation-gradient-descent-1.png)


#### Chain rule 상세.. <a name="chain-rule"></a>

##### 1. `out`을 기준으로한 cost미분은 Logistic regression의 cost function을 미분하는것이다.

![chain-rule-1.png](/images/chain-rule-1.png)

![rogistic-regression-cost-function-differential.png](/images/logistic-regression-cost-function-1.png)

- 그런데, 전체 입력 데이터들에 대한 cost를 계산하는것이 아니라 하나의 instance(tuple)에 대한 cost를 계산하고 그 값으로 backward로 w값을 수정해야 하므로 위 식에서 아래 두가지가 수정되어야 한다.
    - instance index `i`를 모든 instance를 적용할 필요 없으므로 시그마(i)는 의미 없다.
    - 위에서 설명했듯이, multi-class classification일 경우 [Cost function for neural network](#nn-cost-function)처럼 모든 out의 cost를 합해야(Etotal) 하므로 `i`를 모든 `out`의 cost 벡터인 `k`로 변경하자. (편미분하므로 다른 node의 cost는 상수 처리되므로 상관없다.)

##### 2. `net`기준으로 `out`을 미분하면(즉, sigmoid function 미분하면)

![chain-rule-2.png](/images/chain-rule-2.png)

![sigmod-differential-1.png](/images/sigmoid-differential-1.png)

즉, `out`이 0.75136507 이면 

```
0.75136507(1 - 0.75136507) = 0.186815602
```

##### 3. 마지막으로 아래는 걍 편미분 하면 된다.

![chain-rule-3.png](/images/chain-rule-3.png)

#### Cost function for neural network <a name="nn-cost-function"></a>

Chain rule에서 가장 cost가 적은 w값을 찾기 위해 cost function을 이용하여야 한다.
- 최종 output이 하나인 Binary classcification은 Logistic regression의 cost function을 그대로 이용하면 되지만,
일반적으로는 아래처럼 최종 output이 k개인 Multi-class clascification이다. 이것은 모든 output의 cost들을 합해야 한다.

![backpropagation-cost-function-1.png](/images/nn-k-classes.png)

- 즉, Multi-class classcification은 아래처럼 이전에 봤던 Logistic regression의 cost function에서 k개를 합한것이다. (regularization 은 생략하고 설명)

![backpropagation-cost-function-2.png](/images/backpropagation-cost-function-1.png)

### 3. Hidden layer에서 Backward Pass 로 W값을 조정

- Output-layer와 유사하게 w로 Etotal을 chain-rule을 이용하여 미분하면 된다.

![bp-practice-01.png](/images/bp-practice-01.png)

- w1를 미분하는 식은 아래와 같다.
근데 붉은색 박스는 이미 앞에서 계산되어 있으므로 나머지만 계산하면 된다.

![bp-practice-02.png](/images/bp-practice-02.png)

- w1의 값을 조정한다.

![backpropagation-gradient-descent-1.png](/images/backpropagation-gradient-descent-1.png)

- 똑같은 방법으로 모든 Hidden layer의 노드들에 입력되는 모든 w를 수정하면 하나의 instance에 대해 w가 수정된것이다.

