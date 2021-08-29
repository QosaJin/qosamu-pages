---
toc: true
layout: post
description: ベル状態の解説からIBM量子コンピューター実機でのプログラム実行まで
categories: [markdown]
title: IBM Quantum ExperienceとQiskitでベル状態を作って量子コンピューター実機で測定してみる
author: "Kazuki Motohashi"
---
# IBM Quantum ExperienceとQiskitでベル状態を作って量子コンピューター実機で測定してみる

こんにちは。普段は機械学習のお仕事をしていますが、Amazon Braketに出会って量子コンピューターに興味津々になっています。

## はじめに

最近、YouTubeで見れる[慶應の量子コンピューターの集中講義](https://www.youtube.com/watch?v=R2fyLl7KZXM)（日本語）や[2020年11月6日から始まったCERNのセミナー](https://home.cern/news/announcement/computing/online-introductory-lectures-quantum-computing-6-november)（英語）を見て勉強しています。

前者の講義は2005年に開催されたもので昨今の量子コンピューターブームとは関係なく淡々と[Nielsen & Chuangの本](https://www.amazon.co.jp/dp/4274200078)を題材にした理論の解説をしていますが、後者は[IBM Quantum Experience](https://quantum-computing.ibm.com/)や[Quirk](https://algassert.com/quirk)でグラフィカルに量子回路を作りながらイメージしやすい解説がなされています。

扱っている題材自体も一部異なっていますし、後者は特に量子機械学習にも触れる予定のようなので、片方のコンテンツだけでなく相補的に観るといいんじゃないかと思っています。

## ベル状態 (Bell state)

さて、本稿では[CERNセミナーの第二回](https://indico.cern.ch/event/970904/)で扱われていたいわゆるベル状態 (Bell state) を作ってみます。コード自体も基本的にはCERNセミナーで紹介されていた[こちら](https://indico.cern.ch/event/970904/attachments/2142161/3609810/3.-Hello%2C%20entangled%20world%21.ipynb)を利用します。

ベル状態とは何かという話を最初にしておきます。

独立した2つの量子ビットがあるとき、その系の状態は$\vert 00\rangle$, $\vert 01\rangle$, $\vert 10\rangle$, $\vert 11\rangle$という4つの状態を基底にとった空間上で表されるでしょう。

しかし当然基底の取り方は任意性があるので、上記の基底を回転させてそれぞれの状態が絡み合ったものを基底としても良いわけです。 

特に $(\vert 00\rangle + \vert 11\rangle)/\sqrt{2}$, $(\vert 00\rangle - \vert 11\rangle)/\sqrt{2}$, $(\vert 01\rangle + \vert 10\rangle)/\sqrt{2}$, $(\vert 01\rangle - \vert 10\rangle)/\sqrt{2}$ の状態の基底をベル状態と呼びます。

後で見るように、これらの状態はアダマールゲート (Hadamard gate) とCNOT (Controlled NOT) ゲートを組み合わせることで実現できます。

## IBM Quantum Experience

[IBM Quantum Experience](https://quantum-computing.ibm.com/) はIBMが提供する量子コンピュータープラットフォームです。無料で量子コンピューターのシミュレーターや実機にアクセスできる優れものです。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/135044/9e128346-5c15-394b-c312-457e742b85b7.png)

Circuit Composerというサービスを使えば上図のようにGUI上でドラッグアンドドロップするだけでお手軽に量子回路を組み立てられます。また、IBM Quantum ExperienceではQuantum LabというマネージドのJupyter Notebook環境も使えます。ここではQuantum Lab上でQiskit (kiss-kitのように発音) というオープンソースの量子回路開発フレームワークを使ってベル状態を作ってみます。

## Qiskitを用いた量子回路の構築

Qiskitを使って量子回路を構築するのはとても簡単です。まず `qiskit.circuit.QuantumCircuit
` を使って量子ビットと古典ビットの数を指定します。今回は2量子ビットかつ測定用の2古典ビットを用意します。

そして、0番目の量子ビットにアダマールゲート `h` を作用 (`.h(0)`) させ、0番目で制御されるCNOTゲートを1番目の量子ビットに作用 (`.cx(0, 1)`) させます。最後に、0番目の量子ビットを0番目の古典ビットで測定し、1番目の量子ビットを1番目の古典ビットで測定するという宣言 (`.measure((0, 1), (0, 1))`) をしてあげます 。

```python:
%matplotlib inline

from qiskit import QuantumCircuit, execute, Aer, IBMQ
from qiskit.tools.monitor import backend_overview, backend_monitor, job_monitor
from qiskit.visualization import plot_histogram

# 2つの量子ビットと2つの古典ビット（測定用）からなる回路
circ_bell = QuantumCircuit(2, 2)

circ_bell.h(0)  # アダマールゲート
circ_bell.cx(0, 1)  # CNOTゲート
circ_bell.measure((0, 1), (0, 1)) # 測定

circ_bell.draw(output='mpl')  # 回路の描画
```

`.draw(output='mpl')` で描画された回路は以下のようになります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/135044/4670fe9c-9e1e-48ab-6e6e-6bee6b5196bc.png)

こちらの回路の測定を量子コンピューターのシミュレーターと実機で試してみましょう。

## Qiskit Aer シミュレーター

まずは[Qiskit Aer](https://qiskit.org/documentation/tutorials/simulators/1_aer_provider.html)というシミュレーターを使って計算します。

Qiskit Aerでは`QasmSimulator` や `StatevectorSimulator` など利用用途が異なるバックエンドが複数用意されています。`QasmSimulator` は実機を模したシミュレーターであり出力はカウント数などですが、`StatevectorSimulator` では終状態のベクトルを得ることができます。

`QasmSimulator` を使った測定と結果は以下の通りです。

```python:
# QasmSimulatorバックエンドを取得
backend = Aer.get_backend('qasm_simulator')
# circ_bellをQasmSimulatorで1000回測定
job = execute(circ_bell, backend, shots=1000)

# 測定結果の取得
counts = job.result().get_counts()

print(counts)
# >>> {'00': 493, '11': 507}
```

量子コンピューターでは出力は確率的に決まるものであり、一回測定するだけでは求めたい結果が得られません。（もちろん理想的には測定したら必ず0が出力される、といった状態を作ることもできますが、実機ではノイズもあるので結局複数回の測定は必要になるかと思います。）

ここでは1000回同じ測定を行い、両方のビットが0、もしくは1である確率が大体半々であるという結果が得られました。（測定回数（ショット数）をさらに増やせば誤差は小さくなっていくでしょう。）

初期状態が $\vert 00\rangle$ の場合、上記の回路に通すと終状態は $(\vert 00\rangle + \vert 11\rangle)/\sqrt{2}$ になりますので、00と11が半々というのは期待通りの結果が得られたことになります。

## IBM Quantum Experienceで使える量子コンピューター

では、次に量子コンピューター実機で試してみましょう。

その前にIBM Quantum Experienceではどんな量子コンピューターが用意されているかを確認しましょう。

（なお、以下のコードはマネージドJupyter NotebookのQuantum Lab上であれば問題なく動作するはずですが、ローカルで同じコードを実行しようとするとIBM Quantum Experienceのクレデンシャルを求められると思います。）

```python:
provider = IBMQ.load_account()
backend_overview()
```

出力は以下のようになります。それぞれのパラメーターの意味は次のようになります。

- それぞれの量子コンピューターの名前
- 用いることのできる量子ビットの数 (Num. Qubits)
- 現在キューに溜まってるジョブ数 (Pending Jobs)
- 一番空いているか否か (Least busy)
- 運転中か否か (Operational)
- $\vert 1\rangle$ が $\vert 0\rangle$ にdecayしてしまうまでの平均時間 (Avg. T1) [$\mu$s]
- 量子もつれの状態が解消してしまうまでの平均時間 (Avg. T2) [$\mu$s]

```
ibmq_santiago                ibmq_athens                  ibmq_armonk
-------------                -----------                  -----------
Num. Qubits:  5              Num. Qubits:  5              Num. Qubits:  1
Pending Jobs: 25             Pending Jobs: 8              Pending Jobs: 1
Least busy:   False          Least busy:   False          Least busy:   True
Operational:  True           Operational:  True           Operational:  True
Avg. T1:      152.4          Avg. T1:      66.7           Avg. T1:      157.9
Avg. T2:      113.5          Avg. T2:      84.3           Avg. T2:      190.0



ibmq_valencia                ibmq_ourense                 ibmq_vigo
-------------                ------------                 ---------
Num. Qubits:  5              Num. Qubits:  5              Num. Qubits:  5
Pending Jobs: 105            Pending Jobs: 18             Pending Jobs: 53
Least busy:   False          Least busy:   False          Least busy:   False
Operational:  True           Operational:  True           Operational:  True
Avg. T1:      90.0           Avg. T1:      98.2           Avg. T1:      122.8
Avg. T2:      52.9           Avg. T2:      71.8           Avg. T2:      85.1



ibmqx2                       ibmq_16_melbourne
------                       -----------------
Num. Qubits:  5              Num. Qubits:  15
Pending Jobs: 912            Pending Jobs: 1279
Least busy:   False          Least busy:   False
Operational:  True           Operational:  False
Avg. T1:      54.3           Avg. T1:      55.2
Avg. T2:      38.1           Avg. T2:      59.4
```

それぞれの量子コンピューターの仕様の詳細を見るには [`qiskit.tools.backend_monitor`](https://qiskit.org/documentation/stubs/qiskit.tools.backend_monitor.html) を用います。

```python:
backend_monitor(provider.get_backend("ibmq_valencia"))
```

出力は以下のようになります。表示は指定されたバックエンドの名前によって変化します。

```
ibmq_valencia
=============
Configuration
-------------
    n_qubits: 5
    operational: True
    status_msg: active
    pending_jobs: 0
    backend_version: 1.4.0
    basis_gates: ['id', 'u1', 'u2', 'u3', 'cx']
    local: False
    simulator: False
    meas_levels: [1, 2]
    parametric_pulses: ['gaussian', 'gaussian_square', 'drag', 'constant']
    dtm: 2.222222222222222e-19
    credits_required: True
    dt: 2.222222222222222e-19
    max_shots: 8192
    meas_map: [[0, 1, 2, 3, 4]]
    n_registers: 1
    max_experiments: 75
    memory: True
    uchannels_enabled: True
...（以下略）
```

ちなみにIBM Quantum Experienceのトップページの右側にあるバックエンドのリストをクリックすると綺麗なトポロジーのグラフや詳細のパラメーターを確認することもできます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/135044/fb0a4f2c-dfb6-6d4e-5555-84715d43a43c.png)


## 量子コンピューター実機を使った測定

前置きが長くなりましたが、量子コンピューター実機を使った測定をしてみましょう。

上記の Pending Jobs のパラメーターの説明からもわかるように、IBM Quantum Experienceの量子コンピューターは順番待ちで利用するものです。

いちいち空いているバックエンドを探して指定するのは大変ですが、実は `least_busy` という関数を使うと一番空いているバックエンドを選択してくれます。便利ですね。

`provider.backends` に条件を入れて `least_busy` で囲ってあげれば、条件に合うの中から空いているものが選ばれます。

```python:
from qiskit.providers.ibmq import least_busy

# We execute on the least busy device (among the actual quantum computers)
backend = least_busy(
    provider.backends(
        operational=True,  # 稼働中のバックエンド
        simulator=False,   # シミュレーターでなく実機
        status_msg='active',
        filters=lambda x: x.configuration().n_qubits > 1  # 量子ビット数が2以上
    )
)
# 選択されたバックエンドの名前を表示
print("We are executing on...", backend)
# 待ち状態のジョブ数を表示
print("It has", backend.status().pending_jobs, "pending jobs")

job_exp = execute(circ_bell, backend=backend)  # shot数のデフォルトは1,024回
job_monitor(job_exp)
# >>>
# We are executing on... ibmq_athens
# It has 9 pending jobs
# Job Status: job has successfully run
```

今回は `ibmq_athens` という実機の量子コンピューターが選択されました。私の前に9個のジョブが溜まっていましたが、10分程度待ったらジョブが実行されました。

## シミュレーターと実機の比較

最後に実機で得られた測定結果をプロットし、`QasmSimulator` の結果を比較してみます。

```python:
result_exp = job_exp.result()
counts_exp = result_exp.get_counts(circ_bell)
plot_histogram([counts_exp, counts], legend=['Device', 'Simulator'])
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/135044/f8f6539b-19a6-c63c-27b0-67d406f14ec6.png)

横軸は測定結果のビットで、縦軸がそれぞれの確率を表しています。青色が実機で赤色がシミュレーターの測定結果です。

期待通り、量子コンピューター実機でも 00 と 11 の状態が半々であるという結果になりました。
しかし、実機では多少 01 や 10 の結果も含まれてしまっています。これは実機におけるゲート操作のエラーや測定の誤差に起因するものでしょう。
今回は QasmSimulator にノイズの設定をしなかったので、シミュレーター上では理想的な 00/11 のみの状態が再現されていました。

## まとめ

今回はIBM Quantum ExperienceのマネージドJupyter NotebookサービスであるQuantum Lab上で量子回路を構築してみました。

アダマールゲートとCNOTゲートを用いて、ベル状態のひとつ $(\vert 00\rangle + \vert 11\rangle)/\sqrt{2}$ を作る回路をQiskitで構築しました。

シミュレーターと実機で測定をしたところ、出力はほとんど 00 もしくは 11 となり、期待された状態が得られていることがわかりました。

`least_busy` 関数を用いることでIBMが提供する量子コンピューターのうち一番空いているものを取得する方法に関しても解説しました。
