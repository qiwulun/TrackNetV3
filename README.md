`cd ckpts && split -b 90m TrackNet_best.pt partTrack`

 每个分块（当然，最后一个不保证）大小都是50k，基本不可读。任何类型文件都可以用这种切割模式。 三、文件合并 不管用什么方式切割，合并方法不变。 
 
`cd ckpts && cat partTrack* > TrackNet_best.pt`


- 生成视频 `!cd TrackNetV3 && python predict.py --video_file test.mp4 --tracknet_file ckpts/TrackNet_best.pt --inpaintnet_file ckpts/InpaintNet_best.pt --save_dir prediction --output_video`
- 不生成视频 `!cd TrackNetV3 && python predict.py --video_file test.mp4 --tracknet_file ckpts/TrackNet_best.pt --inpaintnet_file ckpts/InpaintNet_best.pt --save_dir prediction`

- 修改 `prediction.py`: 1num_workers = 0 # fixme: 不是0的话在 colab 上面会崩溃 # args.batch_size if args.`

# TrackNetV3
We present TrackNetV3, a model composed of two core modules: trajectory prediction and rectification. The trajectory prediction module leverages an estimated background as auxiliary data to locate the shuttlecock in spite of the fluctuating visual interferences. This module also incorporates mixup data augmentation to formulate complex
scenarios to strengthen the network’s robustness. Given that a shuttlecock can occasionally be obstructed, we create repair masks by analyzing the predicted trajectory, subsequently rectifying the path via inpainting.
[[paper](https://dl.acm.org/doi/10.1145/3595916.3626370)]

<div align="center">
    <a href="./">
        <img src="./figure/NetArch.png" width="50%"/>
    </a>
</div>

## Performance 

* Performance on the test split of [Shuttlecock Trajectory Dataset](https://hackmd.io/Nf8Rh1NrSrqNUzmO0sQKZw).

<div align="center">
    <table>
    <thead>
        <tr>
        <th>Model</th> <th>Accuracy</th> <th>Precision</th> <th>Recall</th> <th>F1</th> <th>FPS</th>
        </tr>
    </thead>
    <tbody>
        <tr>
        <td>YOLOv7</td> <td>57.82%</td> <td>78.53%</td> <td>59.96%</td> <td>68.00%</td> <td><b>34.77</b></td>
        </tr>
        <tr>
        <td>TrackNetV2</td> <td>94.98%</td> <td><b>99.64%</b></td> <td>94.56%</td> <td>97.03%</td> <td>27.70</td>
        </tr>
        <tr>
        <td>TrackNetV3</td> <td><b>97.51%</b></td> <td>97.79%</td> <td><b>99.33%</b></td> <td><b>98.56%</b></td> <td>25.11</td>
        </tr>
    </tbody>
    </table>
    </br>
    <a href="./">
        <img src="./figure/Comparison.png" width="80%"/>
    </a>
</div>

## Installation
* Develop Environment
    ```
    Ubuntu 16.04.7 LTS
    Python 3.8.7
    torch 1.10.0
    ```
* Clone this reposity.
    ```
    git clone https://github.com/qaz812345/TrackNetV3.git
    ```

* Install the requirements.
    ```
    pip install -r requirements.txt
    ```

## Inference
* Download the [checkpoints](https://drive.google.com/file/d/1CfzE87a0f6LhBp0kniSl1-89zaLCZ8cA/view?usp=sharing)
* Unzip the file and place the parameter files to ```ckpts```
    ```
    unzip TrackNetV3_ckpts.zip
    ```
* Predict the label csv from the video
    ```
    python predict.py --video_file test.mp4 --tracknet_file ckpts/TrackNet_best.pt --inpaintnet_file ckpts/InpaintNet_best.pt --save_dir prediction
    ```

* Predict the label csv from the video, and output a video with predicted trajectory
    ```
    python predict.py --video_file test.mp4 --tracknet_file ckpts/TrackNet_best.pt --inpaintnet_file ckpts/InpaintNet_best.pt --save_dir prediction --output_video
    ```

## Training
### 1. Prepare Dataset
* Download [Shuttlecock Trajectory Dataset](https://hackmd.io/Nf8Rh1NrSrqNUzmO0sQKZw)
* Set the data root directory to ```data_dir``` in ```dataset.py```.
* Data Preprocessing
    ```
    python preprocess.py
    ```
* The `frame` directories and the `val` directory will be generated after preprocessing.
* Check the estimated background images in `<data_dir>/median`
    * If available, the dataset will use the median image of the match; otherwise, it will use the median image of the rally.
    * For example, you can exclude `train/match16/median.npz` due to camera angle discrepancies; therefore, the dataset will resort to the median image of the rally within match 16.
* The preprocessed dataset will be cached using npy files, so please ensure that you delete these files if you make any modifications to the dataset.
* Dataset File Structure:
```
  data
    ├─ train
    |   ├── match1/
    |   │   ├── csv/
    |   │   │   ├── 1_01_00_ball.csv
    |   │   │   ├── 1_02_00_ball.csv
    |   │   │   ├── …
    |   │   │   └── *_**_**_ball.csv
    |   │   ├── frame/
    |   │   │   ├── 1_01_00/
    |   │   │   │   ├── 0.png
    |   │   │   │   ├── 1.png
    |   │   │   │   ├── …
    |   │   │   │   └── *.png
    |   │   │   ├── 1_02_00/
    |   │   │   │   ├── 0.png
    |   │   │   │   ├── 1.png
    |   │   │   │   ├── …
    |   │   │   │   └── *.png
    |   │   │   ├── …
    |   │   │   └── *_**_**/
    |   │   │
    |   │   └── video/
    |   │       ├── 1_01_00.mp4
    |   │       ├── 1_02_00.mp4
    |   │       ├── …
    |   │       └── *_**_**.mp4
    |   ├── match2/
    |   │ ⋮
    |   └── match26/
    ├─ val
    |   ├── match1/
    |   ├── match2/
    |   │ ⋮
    |   └── match26/
    └─ test
        ├── match1/
        ├── match2/
        └── match3/
```
* Attributes in each csv files: `Frame, Visibility, X, Y`
### 2. Train Tracking Module
* Train the tracking module from scratch
    ```
    python train.py --model_name TrackNet --seq_len 8 --epochs 30 --batch_size 10 --bg_mode concat --alpha 0.5 --save_dir exp --verbose
    ```

* Resume training (start from the last epoch to the specified epoch)
    ```
    python train.py --model_name TrackNet --epochs 30 --save_dir exp --resume_training --verbose
    ```

### 3. Generate Predited Trajectories and Inpainting Masks
* Generate predicted trajectories and inpainting masks for training rectification module
    * Noted that the coordinate range corresponds to the input spatial dimensions, not the size of the original image.
    ```
    python generate_mask_data.py --tracknet_file ckpts/TrackNet_best.pt --batch_size 16
    ```

### 4. Train Rectification Module
* Train the rectification module from scratch.
    ```
    python train.py --model_name InpaintNet --seq_len 16 --epoch 300 --batch_size 32 --lr_scheduler StepLR --mask_ratio 0.3 --save_dir exp --verbose
    ```

* Resume training (start from the last epoch to the specified epoch)
    ```
    python train.py --model_name InpaintNet --epochs 30 --save_dir exp --resume_training
    ```

## Evaluation
* Evaluate TrackNetV3 on test set
    ```
    python generate_mask_data.py --tracknet_file ckpts/TrackNet_best.pt --split_list test
    python test.py --inpaintnet_file ckpts/InpaintNet_best.pt --save_dir eval
    ```

* Evaluate the tracking module on test set
    ```
    python test.py --tracknet_file ckpts/TrackNet_best.pt --save_dir eval
    ```

* Generate video with ground truth label and predicted result
    ```
    python test.py --tracknet_file ckpts/TrackNet_best.pt --video_file data/test/match1/video/1_05_02.mp4 
    ```

## Error Analysis Interface
* Evaluate TrackNetV3 on test set and save the detail results for error analysis
    ```
    python test.py --tracknet_file ckpts/TrackNet_best.pt --inpaintnet_file ckpts/InpaintNet_best.pt --save_dir eval --output_pred
    ```

* Add json path of evaluation results to the file list in `error_analysis.py`
    ```
    30  # Evaluation result file list
    31  if split == 'train':
    32      eval_file_list = [
    33          {'label': label_name, 'value': json_path},
     ⋮                              ⋮
            ]
        elif split == 'val':
            eval_file_list = [
                {'label': label_name, 'value': json_path},
                                    ⋮
            ]
        elif split == 'test':
            eval_file_list = [
                {'label': label_name, 'value': json_path},
                                    ⋮
            ]
        else:
            raise ValueError(f'Invalid split: {split}')                                  
    ```

* Run Dash application
    ```
    python error_analysis.py --split test --host 127.0.0.1
    ```
<div align="center">
    <a href="./">
        <img src="./figure/ErrorAnalysisUI.png" width="70%"/>
    </a>
</div>
