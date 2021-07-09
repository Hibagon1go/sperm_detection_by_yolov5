# dockerによる環境構築で、yolov5を学習させる方法

## ファイル構成(このReadme.mdのrawファイルを見てください)
sperm_eval/

　├ Dockerfile
 
　├ sperm.yaml
 
　├ sperm/
 
　│　└ sperm_train
 
　│　└ sperm_val
 
　│　└ sperm_test
 
　└ results/
 
　　 └ 適宜結果を入れるファイル作成

## Dockerfile

FROM python:3.8.0

RUN apt-get update \
  && apt-get install -y git curl wget make libgl1-mesa-dev \
  && apt clean

EXPOSE 8889

WORKDIR /home/work

RUN git clone https://github.com/ultralytics/yolov5 \
  && pip install --upgrade pip \
  &&pip install -r yolov5/requirements.txt

COPY sperm/ yolov5/sperm/
COPY sperm.yaml yolov5/sperm.yaml

## sperm.yaml

train: sperm/sperm_train

val: sperm/sperm_val

test: sperm/sperm_test

nc: 1

names: ['sperm']

## 学習の流れ

1. Dockerfileと同じ階層で、docker build . (--no-cache)によりdocker image作成
2. docker imagesで、今作ったimageのIDをコピー
3. docker run -it --shm-size=8gb --gpus 1 [ImageID] [command(/bin/bashとか)] で、shared-memoryの容量増やし、かつGPU使いながら、コンテナに入る
※ 初回時exitすると何故かコンテナがストップことがある?が、docker start [continerID] でもう一回走らせ、docker exec -it [containerID] [command(/bin/bashとか)]で入るべし

4. cd yolov5でyolov5に入り、python train.py --cfg yolov5s(or m or l or x).yaml --img 742 742 --batch 20 --epochs 300 --data sperm.yaml --name sperm (--device 0) で学習。その結果はyolov5/runs/train/spermにある。
5. python detect.py --weights runs/train/sperm/weights/best.pt --source sperm/sperm_test (--device 0)で、画像に対し実際に推論。その結果はyolov5/runs/detect/expにある。
6. python test.py --data sperm.yaml --weights runs/train/sperm/weights/best.pt --task testで、テストデータに対する汎化性能を見る。その結果はyolov5/runs/test/expにある。

※ RecallとPrecisionとかそこらへんの結果を重ねて表示したいときは、階層yolov5で、mv yolov5/runs/train/sperm/results.txt . とし、results.txtを階層yolov5にまず持ってくる。そこで、pythonとうち、インタラクションモードに入り、その中で from utils.plots import * (Enterのあとに)plot_results_overlay()で実行。yolov5階層にresults.pngができる。

7. 結果をコンテナ外に持ってくる。results/適宜結果を入れるファイル作成 に入り、そこで docker cp [containerID+持ってきたい結果までのパス(4d7f660c44dd:/home/work/yolov5/run/detect/expとか)] . で持って来れる。

