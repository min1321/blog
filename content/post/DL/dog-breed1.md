---
title: "강아지 분류 인공지능 웹사이트 만들기 (1)"
date: 2021-11-28T21:17:32+09:00
categories:
- DeepLeaning
- ImageClassification
tags:
- 강아지 품종 분류
- DeepLeaning
- ImageClassification
- CV
- dog breed
keywords:
- Dog Breed dataset
thumbnailImage: //min1321.github.io/images/dogbreed/Chihuahua.jpg
---

<!--more-->
![chihuahua](/images/dogbreed/Chihuahua.jpg)
강아지 품종 인공지능 분류 웹 사이트를 만들어 보려한다.

한번에 하기에 양이 많아서 아래 처럼 post를 나눠서 올리겠다.
1. 강아지 데이터 다운 및 loader 정의하기
2. 강아지 분류 모델 학습
3. AWS EC2 사용해서 Web 만들기

## Dataset Down & Data Split
---
Dataset은 스텐포드 dogs dataset을 사용하였다.
```
# make dog breed project directory
$ mkdir dog-breed

# dataset download
$ wget http://vision.stanford.edu/aditya86/ImageNetDogs/images.tar
```

이제 주피터 노트북으로 folder내 이미지들 경로를 학습하기 좋게 바꾸어 주었다.
```python
# tar 파일 풀기
!tar -xvf images.tar
```

```python
# 필요한 파일들 import
import glob
import shutil
from collections import defaultdict
import os
```

```python
!pwd
#/root/works/workspace/dog-breed
```

```python
file_list = glob.glob('./Images/*/*')
len(file_list)
# 전체 이미지 수 20580
```

파일명과 폴더명을 보고 어떻게 옮겨 담을지 확인한다.
강아지 품종 명은 폴더의 `-` 기준으로 분리되어있다.

```python
file_list[0]
```

    './Images/n02085620-Chihuahua/n02085620_10621.jpg'




```python
breed_dict = defaultdict(list)
for file in file_list:
    breed = "-".join(file.split('/')[2].split('-')[1:])
    file_name = file.split('/')[-1]
    breed_dict[breed].append(file_name)
```


```python
# 품종 개수
len(breed_dict)
# 120
```

Train Valid Test Set을 분리하기 위해 폴더를 만들어준다.
```python
os.mkdir("./data")
os.mkdir("./data/train")
os.mkdir("./data/valid")
os.mkdir("./data/test")

for key in breed_dict:
    os.mkdir("./data/test/"+key)
    os.mkdir("./data/valid/"+key)
    os.mkdir("./data/train/"+key)
```


여기서 Train 70%, Valid 20%, Test 10%를 정해주었다. 여기서 validation set으로 평가하면 충분한데 왜 test set을 다시 분리해서 평가할 필요가 있을까? 
왜냐하면 validation set은 학습에 사용되지는 않지만, 학습에 관여는 된다. 왜냐하면, 개발자는 validation Acc를 높이고자 모델을 튜닝할 것이고, 결국 valid set은 학습에 직접적인 사용이 되지는 않지만 영향을 미친다. 그래서 완전히 분리된 test set을 두어 오직 '최종 성능'을 평가하기 위해 쓰인다.

```python
# 70% train, 20% valid, 10% test

breed_dict = defaultdict(list)
for file in file_list:
    breed = file.split('/')[2].split('-')[1]
    file_name = file.split('/')[-1]
    breed_dict[breed].append(file)
```


```python
train_count = defaultdict(int)
valid_count = defaultdict(int)
test_count = defaultdict(int)
for breed in breed_dict:
    for file in breed_dict[breed]:
        if train_count[breed] < int(len(breed_dict[breed])*0.7):
            train_count[breed] += 1
            shutil.move(file, "./data/train/" + breed + '/' + file.split('/')[-1])
        elif valid_count[breed] < int(len(breed_dict[breed])*0.2):
            valid_count[breed] += 1
            shutil.move(file, "./data/valid/" + breed + '/' + file.split('/')[-1])
        else:
            test_count[breed] += 1
            shutil.move(file, "./data/test/" + breed + '/' + file.split('/')[-1])
```

Class 별 숫자를 확인해서 너무 적은 data를 갖는 class를 확인해보자.
모든 class는 0.7%~ 1.2% 수준으로 너무 부족한 class는 없는 것을 확인하였다.
```python
for key in train_count:
    print(f'{key} : {train_count[key]/14406 :4.2%}')
```

    Chihuahua : 0.74%
    Japanese_spaniel : 0.90%
    Maltese_dog : 1.22%
    Pekinese : 0.72%
    Shih : 1.03%
    ........
    ........
    
 ## Datset 정의하기
 ---
 dataset을 정의할때 `__init__` `__len__` `__getitem__`  을 정의해 줘야한다.
 `__init__`은 dataset이 생성될 때 정의할 부분을 지정해주고, `__len__`에서는 Dataset의 길이를 정의해준다. 마지막으로 `__getitem__` 에서는 Loader에서 index로 input data를 불러올 때 어떤 데이터를 어떤 방식으로 불러올지를 정의해둔 메소드이다.
 ```python
from torch.utils.data import Dataset
import os
import numpy as np
import cv2
import torchvision.transforms as transforms
import cv2
import glob

class CustomDataset(Dataset):
    def __init__(self, path, mode='train', transform=None, dataset="DogBreedDataset"):
        super().__init__()
        self.dataset = dataset
        self.path = path  # data/test/classes/filename, data/train/classes/filename
        self.mode = mode
        self.transform = transform
        if mode == 'train':
            self.path = os.path.join(path, "train")
        elif mode == 'valid':
            self.path = os.path.join(path, "valid")
        elif mode == 'test':
            self.path = os.path.join(path, "test")

        self.categorys = []
        for category_path in glob.glob(self.path + "/*"):
            self.categorys.append(category_path.split('/')[-1])
        self.categorys = sorted(self.categorys)
        
        self.image_files = glob.glob(self.path + "/*/*")   # list
        self.num_classes = len(self.categorys)
        
    def __len__(self):
        return len(self.image_files)

    def __getitem__(self, index):
        image_path = self.image_files[index]
        image = cv2.imread(image_path)
        image = cv2.cvtColor(image, cv2.COLOR_BGR2RGB).astype(np.float32)
        
        if self.mode == 'train' or self.mode == 'valid':
            label_name = image_path.split('/')[-2]
            label_id = self.categorys.index(label_name)
            if self.transform is not None:
                transformed = self.transform(image=image)
                image = transformed["image"]
            return image, label_id
        if self.mode == 'test':
            if self.transform is not None:
                transformed = self.transform(image=image)
                image = transformed["image"]
            return image
 ```
 
 이렇게 정의해둔 Dataset으로 loader를 생성시키는 부분을 보겠다.
 Train transform은 resize와 randomCorp부분만 추가하였고, Normalize는 `Imagenet`에서 사용된 mean과 std를 사용해서 정의해 주었다. 사실 나중에 모델에 `batch normalize`가 들어가기 때문에 의미는 많이 줄어들었다.
 Dataloader에서 sampler는 분산학습을 할 경우 distributed를 True로 학습한다.
 
```python
from torch.utils.data import DataLoader, sampler
import albumentations as A
from albumentations.pytorch import ToTensorV2

train_trans = A.Compose(
    [
        A.Resize(320,320),
        A.RandomCrop(320, 320),
        A.Normalize(mean=(0.485, 0.456, 0.406), std=(0.229, 0.224, 0.225)),
        ToTensorV2(),
    ]
)

train_set = CustomDataset(args.path, mode="train", transform=train_trans, dataset=args.dataset)

train_loader = DataLoader(
    train_set,
    batch_size=args.batch,
    num_workers=1,
    sampler=data_sampler(train_set, shuffle=True, distributed=args.distributed),
)
```

__이후 model을 선정하고, hyper parameter를 튜닝하는 것을 진행하겠다.
2부에서 다시 이어서 포스팅 하도록 하겠다.__