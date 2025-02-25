﻿# ncnn_android_fall_detect


## 流程

1. 開啟Anaconda Prompt
2. ``` $ conda create -n pt2onnxyolonormal python=3.9 ```
3. ``` $ conda activate pt2onnxyolonormal ```
4. ``` $ pip install roboflow ```
5. ``` $ pip install ultralytics ```
6. ``` $ cd C:\Users\user\anaconda3\envs\pt2onnxyolonormal\Lib\site-packages\ultralytics\nn\modules ```
7. 修改 block.py (for yolo normal)
```
class C2f(nn.Module):
    """CSP Bottleneck with 2 convolutions."""

    def __init__(self, c1, c2, n=1, shortcut=False, g=1, e=0.5):  # ch_in, ch_out, number, shortcut, groups, expansion
        super().__init__()
        self.c = int(c2 * e)  # hidden channels
        self.cv1 = Conv(c1, 2 * self.c, 1, 1)
        self.cv2 = Conv((2 + n) * self.c, c2, 1)  # optional act=FReLU(c2)
        self.m = nn.ModuleList(Bottleneck(self.c, self.c, shortcut, g, k=((3, 3), (3, 3)), e=1.0) for _ in range(n))

    def forward(self, x):
        """Forward pass of a YOLOv5 CSPDarknet backbone layer."""
        # y = list(self.cv1(x).chunk(2, 1))
        # y.extend(m(y[-1]) for m in self.m)
        # return self.cv2(torch.cat(y, 1))

        x = self.cv1(x)
        x = [x, x[:, self.c:, ...]]
        x.extend(m(x[-1]) for m in self.m)
        x.pop(1)
        return self.cv2(torch.cat(x, 1))

```
8. 修改 head.py (for yolo normal)  這邊是修改detect的, head.py裡面還有seg, pose等等
```
def forward(self, x):
        """Concatenates and returns predicted bounding boxes and class probabilities."""
        shape = x[0].shape  # BCHW
        for i in range(self.nl):
            x[i] = torch.cat((self.cv2[i](x[i]), self.cv3[i](x[i])), 1)
        if self.training:
            return x
        elif self.dynamic or self.shape != shape:
            self.anchors, self.strides = (x.transpose(0, 1) for x in make_anchors(x, self.stride, 0.5))
            self.shape = shape

        # x_cat = torch.cat([xi.view(shape[0], self.no, -1) for xi in x], 2)
        # if self.export and self.format in ('saved_model', 'pb', 'tflite', 'edgetpu', 'tfjs'):  # avoid TF FlexSplitV ops
        #     box = x_cat[:, :self.reg_max * 4]
        #     cls = x_cat[:, self.reg_max * 4:]
        # else:
        #     box, cls = x_cat.split((self.reg_max * 4, self.nc), 1)
        # dbox = dist2bbox(self.dfl(box), self.anchors.unsqueeze(0), xywh=True, dim=1) * self.strides
        # y = torch.cat((dbox, cls.sigmoid()), 1)
        # return y if self.export else (y, x)

        pred = torch.cat([xi.view(shape[0], self.no, -1) for xi in x], 2).permute(0, 2, 1)
        return pred

```

9. ``` cd C:\Users\user\Downloads\OtherProject\DeepLearning\ultralytics_byAnson\examples\YOLOv8-PTToONNX ```
10. 執行main.py --> 產生onnx 檔案
11. ``` cd C:\Users\user\Downloads\OtherProject\DeepLearning\ncnn_windows\ncnn\build\tools\onnx ```
12. ``` .\onnx2ncnn.exe C:\Users\user\Downloads\OtherProject\DeepLearning\ultralytics_byAnson\examples\YOLOv8-PTToONNX\best.onnx fall.param fall.bin  ``` --> 轉換成ncnn格式


## Seg 轉換

1. 首先是修改block.py中的c2f的forward (for yolo seg)
```     def forward(self, x):
        # """Forward pass through C2f layer."""
        # y = list(self.cv1(x).chunk(2, 1))
        # y.extend(m(y[-1]) for m in self.m)
        # return self.cv2(torch.cat(y, 1))
        # !< https://github.com/FeiGeChuanShu/ncnn-android-yolov8
        x = self.cv1(x)
        x = [x, x[:, self.c:, ...]]
        x.extend(m(x[-1]) for m in self.m)
        x.pop(1)
        return self.cv2(torch.cat(x, 1))
```
2. 然后是修改head.py中的Detect的forward (for yolo seg)
```     def forward(self, x):
        """Concatenates and returns predicted bounding boxes and class probabilities."""
        shape = x[0].shape  # BCHW
        for i in range(self.nl):
            x[i] = torch.cat((self.cv2[i](x[i]), self.cv3[i](x[i])), 1)
        if self.training:
            return x
        elif self.dynamic or self.shape != shape:
            self.anchors, self.strides = (x.transpose(0, 1) for x in make_anchors(x, self.stride, 0.5))
            self.shape = shape
 
        x_cat = torch.cat([xi.view(shape[0], self.no, -1) for xi in x], 2)
        return x_cat
        # if self.export and self.format in ('saved_model', 'pb', 'tflite', 'edgetpu', 'tfjs'):  # avoid TF FlexSplitV ops
        #     box = x_cat[:, :self.reg_max * 4]
        #     cls = x_cat[:, self.reg_max * 4:]
        # else:
        #     box, cls = x_cat.split((self.reg_max * 4, self.nc), 1)
        # dbox = dist2bbox(self.dfl(box), self.anchors.unsqueeze(0), xywh=True, dim=1) * self.strides
 
        # if self.export and self.format in ('tflite', 'edgetpu'):
        #     # Normalize xywh with image size to mitigate quantization error of TFLite integer models as done in YOLOv5:
        #     # https://github.com/ultralytics/yolov5/blob/0c8de3fca4a702f8ff5c435e67f378d1fce70243/models/tf.py#L307-L309
        #     # See this PR for details: https://github.com/ultralytics/ultralytics/pull/1695
        #     img_h = shape[2] * self.stride[0]
        #     img_w = shape[3] * self.stride[0]
        #     img_size = torch.tensor([img_w, img_h, img_w, img_h], device=dbox.device).reshape(1, 4, 1)
        #     dbox /= img_size
 
        # y = torch.cat((dbox, cls.sigmoid()), 1)
        # return y if self.export else (y, x)
```
3.  最后是修改segment的forward (for yolo seg)
```  def forward(self, x):
        """Return model outputs and mask coefficients if training, otherwise return outputs and mask coefficients."""
        p = self.proto(x[0])  # mask protos
        bs = p.shape[0]  # batch size
 
        mc = torch.cat([self.cv4[i](x[i]).view(bs, self.nm, -1) for i in range(self.nl)], 2)  # mask coefficients
        x = self.detect(self, x)
        if self.training:
            return x, mc, p
        # return (torch.cat([x, mc], 1), p) if self.export else (torch.cat([x[0], mc], 1), (x[1], mc, p))
        # !< https://github.com/FeiGeChuanShu/ncnn-android-yolov8
        return (torch.cat([x, mc], 1).permute(0, 2, 1), p.view(bs, self.nm, -1)) if self.export else (torch.cat([x[0], mc], 1), (x[1], mc, p))
```


## 展望
1. yolo fall detect by boundingbox 待整合
2. yolo pose fall detect 待整合
3. yolo alphapose stgcn 待整合

## Reference
1. https://blog.csdn.net/fisherisfish/article/details/134123826 (yolo seg)
2. https://github.com/FeiGeChuanShu/ncnn-android-yolov8 (yolo normal)



