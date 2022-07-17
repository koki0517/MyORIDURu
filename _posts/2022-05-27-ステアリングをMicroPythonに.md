---
layout: post
title: "LEGOステアリングを再現してみよう。MicroPythonに"
date: 2022-05-27
excerpt: "MicroPythonにステアリングを再現してみた。"
tags: []
---

# LEGOステアリングを再現してみよう。MicroPythonに

# はじめに
**ステアリング**とは2つのモーターで走行する際にその向きを-100~100の数で設定する機能です。EV3-G(教育版EV3ソフトウェア)での詳細は[公式ドキュメント](https://ev3-help-online.api.education.lego.com/Retail/ja-jp/page.html?Path=blocks%2FLEGO%2FMove.html)を御覧ください。<br>

<img width="500" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2449798/c4aac323-f393-4150-a290-5aea7a795545.png">

EV3のMicroPythonにEV3-Gのステアリングにあたる機能がないことを知り、自力でステアリングを実装しようと考え、色々調査&検証してみました。※SPIEK PrimeのMicroPythonには標準でステアリング機能があります。EV3のMicroPythonにも似た機能はあります。[参照](https://pybricks.com/ev3-micropython/robotics.html)

# 左右がわからない時
ロボットはステアリング量が0の時真っすぐ進み、-100の時その場で左回転し、100だとその場で右回転しましす。<br>
たまに正負のどちらが左方向、右方向なのかわからづらくなりますが、そんなときは右向きの数直線上にステアリング量を当てはめ、それが0より小さい。つまり数直線の左側にあるときは左に進み、0より大きい。つまり数直線の右側にあるときは右側に進むと考えるとわかりやすいです。<br>

# ステアリング動作を数式で表す
ステアリングの値を$x$とすると、引数の1つであるパワーを$z$、$x$の値によって以下のように動きが変化します。<br>
| ステアリング | 左モーターの出力 | 右モーターの出力 | 動作 |
| - | - | - | - |
| -100 | -$z$ | $z$ | その場で左回転 |
| -50 | 0 | $z$ | 左スピン |
| 0 | $z$ | $z$ | 直進 |
| 50 | $z$ | 0 | 右スピン |
| 100 | $z$ | -$z$ | その場で右回転 |

※2つのモーターで左右のモーターを制御する四輪駆動を想定しています。<br>

この表を元に左モーターの出力を$y_{l}$、右モーターの出力を$y_{r}$とすると<br>
$-100 \leqq x \lt 0$の時、<br>
$y_{l}= \frac{z}{50}x+z$ …①<br>
$y_{r}=z$ …②<br>

$0 \leqq x \leqq 100$の時、<br>
$y_{l}= z$ …③<br>
$y_{r}=-\frac{z}{50}x+z$ …④<br>

という式で表すことができます。<br>
グラフの形は$z =100$のとき、<br>

<img width="600" alt="aa" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2449798/e6faecca-d5d5-0124-7fbd-1cbe1133d549.png">

※横軸が$x$、縦軸が$y$軸、青色のグラフが左モーターの出力、赤色のグラフが右モーターの出力です。<br>

$z$が0に近づくほど2つのグラフは$y=0$に近くなり、$z=0$を機に逆転し下の$z=-100$のような形のグラフになります。<br>

<img width="600" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2449798/48266116-a046-ab27-50bd-22e50e7827eb.png">

# コードに起こす
EV3-Gのステアリング機能をほぼそのままMicroPythonに移植したコードです。

```sample.py
#!/usr/bin/env pybricks-micropython
from pybricks.ev3devices import Motor
from pybricks.parameters import Port, Direction
from pybricks.tools import wait, StopWatch

left_motor = Motor(Port.A)
right_motor = Motor(Port.B)

watch = StopWatch()

def SpeedPercent(percentage):
    return 990 * percentage / 100
    # percentageをdeg/sに変換します
    # Lモーターの最大回転数が960~1020deg/sなのでその平均の990を最大回転数(deg/s)として採用しています。

class Tank:
    def steering(self,power,steering):
        if -100 > power or 100 < power:
            raise ValueError
        if -100 <= steering < 0:
            left_motor.run(SpeedPercent((power / 50) * steering + power)) # 式①
            right_motor.run(SpeedPercent(power)) # 式②
        elif 0 <= steering <= 100:
            left_motor.run(SpeedPercent(power)) # 式③
            right_motor.run(SpeedPercent(-1 * (power / 50) * steering + power)) # 式④
        else:
            raise ValueError

    def steering_for_seconds(self,power,steering,seconds,stop_type = "hold"):
        if seconds <= 0:
            raise ValueError
        time_run = watch.time() + seconds
        while watch.time() <= time_run:
            self.steering(power,steering)
        self.stop(stop_type)

    def steeing_for_degrees(self,power,steering,degrees,stop_type = "hold"):
        if degrees < 0:
            degrees, steering *= -1
        left_angle = left_motor.angle()
        right_angle = right_motor.angle()
        while not abs(left_angle - left_motor.angle()) > degrees or abs(right_angle - right_motor.angle()) > degrees:
            self.steering(power,steering)
        self.stop(stop_type)
        # EV3-Gにおけるステアリングの「角度」「回転数」モードは左右2つのモーターのどちらかの変化量が指定した角度や回転数を超えるまで走るという内容になります。

    def steering_for_rotations(self,power,steering,rotations,stop_type = "hold"):
        self.steeing_for_degrees(power,steering,rotations * 360)
        self.stop(stop_type)

    def stop(self,stop_type):
        if stop_type == "stop":
            left_motor.stop()
            right_motor.stop()
        elif stop_type == "brake":
            left_motor.brake()
            right_motor.brake()
        elif stop_type == "hold":
            left_motor.hold()
            right_motor.hold()
        else:
            raise ValueError

tank = Tank()

# 50%のパワーで10秒間右旋回する
tank.steering_for_seconds(50,100,10)

# 40%のパワーで1024度モーターが回転するまで右スピン
tank.steeing_for_degrees(40,50,1024)

# 70%のパワーでモーターが3回転するまで左斜め前に走る
tank.steering_for_rotations(70,-20,3)
```

※まだまだ初心者なので美しいコードではないかもしれません。エラー処理もしてたりしてなかったり、、、<br>

# 参考文献
著:上田 悦子・小枝 正直・中村 恭之,[これからのロボットプログラミング入門 Pythonで動かすMINDSTORMS EV3](https://www.google.co.jp/books/edition/%E3%81%93%E3%82%8C%E3%81%8B%E3%82%89%E3%81%AE%E3%83%AD%E3%83%9C%E3%83%83%E3%83%88%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9/YNnbDwAAQBAJ?hl=ja&gbpv=0), 講談社, 2020年