<!--lint disable list-item-indent-->
<!--lint disable list-item-spacing-->
<!--lint disable emphasis-marker-->
<!--lint disable maximum-line-length-->
<!--lint disable list-item-spacing-->

# Dataset and annotation formats

## Contents

- [How to add a format](#how-to-add)
- [Format descriptions](#formats)
  - [CVAT](#cvat)
  - [Datumaro](#datumaro)
  - [LabelMe](#labelme)
  - [MOT](#mot)
  - [MOTS](#mots)
  - [COCO](#coco)
  - [PASCAL VOC and mask](#voc)
  - [YOLO](#yolo)
  - [TF detection API](#tfrecord)
  - [ImageNet](#imagenet)
  - [CamVid](#camvid)
  - [WIDER Face](#widerface)
  - [VGGFace2](#vggface2)
  - [Market-1501](#market1501)
  - [ICDAR13/15](#icdar)

## How to add a new annotation format support<a id="how-to-add"></a>

1. Add a python script to `dataset_manager/formats`
1. Add an import statement to [registry.py](./registry.py).
1. Implement some importers and exporters as the format requires.

Each format is supported by an importer and exporter.

It can be a function or a class decorated with
`importer` or `exporter` from [registry.py](./registry.py). Examples:

```python
@importer(name="MyFormat", version="1.0", ext="ZIP")
def my_importer(file_object, task_data, **options):
  ...

@importer(name="MyFormat", version="2.0", ext="XML")
class my_importer(file_object, task_data, **options):
  def __call__(self, file_object, task_data, **options):
    ...

@exporter(name="MyFormat", version="1.0", ext="ZIP"):
def my_exporter(file_object, task_data, **options):
  ...
```

Each decorator defines format parameters such as:

- _name_

- _version_

- _file extension_. For the `importer` it can be a comma-separated list.
  These parameters are combined to produce a visible name. It can be
  set explicitly by the `display_name` argument.

Importer arguments:

- _file_object_ - a file with annotations or dataset
- _task_data_ - an instance of `TaskData` class.

Exporter arguments:

- _file_object_ - a file for annotations or dataset

- _task_data_ - an instance of `TaskData` class.

- _options_ - format-specific options. `save_images` is the option to
  distinguish if dataset or just annotations are requested.

[`TaskData`](../bindings.py) provides many task properties and interfaces
to add and read task annotations.

Public members:

- **TaskData. Attribute** - class, `namedtuple('Attribute', 'name, value')`

- **TaskData. LabeledShape** - class, `namedtuple('LabeledShape', 'type, frame, label, points, occluded, attributes, group, z_order')`

- **TrackedShape** - `namedtuple('TrackedShape', 'type, points, occluded, frame, attributes, outside, keyframe, z_order')`

- **Track** - class, `namedtuple('Track', 'label, group, shapes')`

- **Tag** - class, `namedtuple('Tag', 'frame, label, attributes, group')`

- **Frame** - class, `namedtuple('Frame', 'frame, name, width, height, labeled_shapes, tags')`

- **TaskData. shapes** - property, an iterator over `LabeledShape` objects

- **TaskData. tracks** - property, an iterator over `Track` objects

- **TaskData. tags** - property, an iterator over `Tag` objects

- **TaskData. meta** - property, a dictionary with task information

- **TaskData. group_by_frame()** - method, returns
  an iterator over `Frame` objects, which groups annotation objects by frame.
  Note that `TrackedShape` s will be represented as `LabeledShape` s.

- **TaskData. add_tag(tag)** - method,
  tag should be an instance of the `Tag` class

- **TaskData. add_shape(shape)** - method,
  shape should be an instance of the `Shape` class

- **TaskData. add_track(track)** - method,
  track should be an instance of the `Track` class

Sample exporter code:

```python
...
# dump meta info if necessary
...
# iterate over all frames
for frame_annotation in task_data.group_by_frame():
  # get frame info
  image_name = frame_annotation.name
  image_width = frame_annotation.width
  image_height = frame_annotation.height
  # iterate over all shapes on the frame
  for shape in frame_annotation.labeled_shapes:
    label = shape.label
    xtl = shape.points[0]
    ytl = shape.points[1]
    xbr = shape.points[2]
    ybr = shape.points[3]
    # iterate over shape attributes
    for attr in shape.attributes:
      attr_name = attr.name
      attr_value = attr.value
...
# dump annotation code
file_object.write(...)
...
```

Sample importer code:

```python
...
#read file_object
...
for parsed_shape in parsed_shapes:
  shape = task_data.LabeledShape(
    type="rectangle",
    points=[0, 0, 100, 100],
    occluded=False,
    attributes=[],
    label="car",
    outside=False,
    frame=99,
  )
task_data.add_shape(shape)
```

## Format specifications<a id="formats" />

### CVAT<a id="cvat" />

This is the native CVAT annotation format. It supports all CVAT annotations
features, so it can be used to make data backups.

- supported annotations: Rectangles, Polygons, Polylines,
  Points, Cuboids, Tags, Tracks

- attributes are supported

- [Format specification](/cvat/apps/documentation/xml_format.md)

#### CVAT for images export

Downloaded file: a ZIP file of the following structure:

```bash
taskname.zip/
├── images/
|   ├── img1.png
|   └── img2.jpg
└── annotations.xml
```

- tracks are split by frames

#### CVAT for videos export

Downloaded file: a ZIP file of the following structure:

```bash
taskname.zip/
├── images/
|   ├── frame_000000.png
|   └── frame_000001.png
└── annotations.xml
```

- shapes are exported as single-frame tracks

#### CVAT loader

Uploaded file: an XML file or a ZIP file of the structures above

### Datumaro format <a id="datumaro" />

[Datumaro](https://github.com/openvinotoolkit/datumaro/) is a tool, which can
help with complex dataset and annotation transformations, format conversions,
dataset statistics, merging, custom formats etc. It is used as a provider
of dataset support in CVAT, so basically, everything possible in CVAT
is possible in Datumaro too, but Datumaro can offer dataset operations.

- supported annotations: any 2D shapes, labels
- supported attributes: any

### [Pascal VOC](http://host.robots.ox.ac.uk/pascal/VOC/)<a id="voc" />

- [Format specification](http://host.robots.ox.ac.uk/pascal/VOC/voc2012/devkit_doc.pdf)

- supported annotations:

  - Rectangles (detection and layout tasks)
  - Tags (action- and classification tasks)
  - Polygons (segmentation task)

- supported attributes:

  - `occluded` (both UI option and a separate attribute)
  - `truncated` and `difficult` (should be defined for labels as `checkbox` -es)
  - action attributes (import only, should be defined as `checkbox` -es)
  - arbitrary attributes (in the `attributes` secion of XML files)

#### Pascal VOC export

Downloaded file: a zip archive of the following structure:

```bash
taskname.zip/
├── JPEGImages/
│   ├── <image_name1>.jpg
│   ├── <image_name2>.jpg
│   └── <image_nameN>.jpg
├── Annotations/
│   ├── <image_name1>.xml
│   ├── <image_name2>.xml
│   └── <image_nameN>.xml
├── ImageSets/
│   └── Main/
│       └── default.txt
└── labelmap.txt

# labelmap.txt
# label : color_rgb : 'body' parts : actions
background:::
aeroplane:::
bicycle:::
bird:::
```

#### Pascal VOC import

Uploaded file: a zip archive of the structure declared above or the following:

```bash
taskname.zip/
├── <image_name1>.xml
├── <image_name2>.xml
└── <image_nameN>.xml
```

It must be possible for CVAT to match the frame name and file name
from annotation `.xml` file (the `filename` tag, e. g.
`<filename>2008_004457.jpg</filename>` ).

There are 2 options:

1. full match between frame name and file name from annotation `.xml`
   (in cases when task was created from images or image archive).

1. match by frame number. File name should be `<number>.jpg`
   or `frame_000000.jpg`. It should be used when task was created from video.

#### Segmentation mask export

Downloaded file: a zip archive of the following structure:

```bash
taskname.zip/
├── labelmap.txt # optional, required for non-VOC labels
├── ImageSets/
│   └── Segmentation/
│       └── default.txt # list of image names without extension
├── SegmentationClass/ # merged class masks
│   ├── image1.png
│   └── image2.png
└── SegmentationObject/ # merged instance masks
    ├── image1.png
    └── image2.png

# labelmap.txt
# label : color (RGB) : 'body' parts : actions
background:0,128,0::
aeroplane:10,10,128::
bicycle:10,128,0::
bird:0,108,128::
boat:108,0,100::
bottle:18,0,8::
bus:12,28,0::
```

Mask is a `png` image with 1 or 3 channels where each pixel
has own color which corresponds to a label.
Colors are generated following to Pascal VOC [algorithm](http://host.robots.ox.ac.uk/pascal/VOC/voc2012/htmldoc/devkit_doc.html#sec:voclabelcolormap).
`(0, 0, 0)` is used for background by default.

- supported shapes: Rectangles, Polygons

#### Segmentation mask import

Uploaded file: a zip archive of the following structure:

```bash
  taskname.zip/
  ├── labelmap.txt # optional, required for non-VOC labels
  ├── ImageSets/
  │   └── Segmentation/
  │       └── <any_subset_name>.txt
  ├── SegmentationClass/
  │   ├── image1.png
  │   └── image2.png
  └── SegmentationObject/
      ├── image1.png
      └── image2.png
```

It is also possible to import grayscale (1-channel) PNG masks.
For grayscale masks provide a list of labels with the number of lines equal
to the maximum color index on images. The lines must be in the right order
so that line index is equal to the color index. Lines can have arbitrary,
but different, colors. If there are gaps in the used color
indices in the annotations, they must be filled with arbitrary dummy labels.
Example:

```
q:0,128,0:: # color index 0
aeroplane:10,10,128:: # color index 1
_dummy2:2,2,2:: # filler for color index 2
_dummy3:3,3,3:: # filler for color index 3
boat:108,0,100:: # color index 3
...
_dummy198:198,198,198:: # filler for color index 198
_dummy199:199,199,199:: # filler for color index 199
...
the last label:12,28,0:: # color index 200
```

- supported shapes: Polygons

#### How to create a task from Pascal VOC dataset

1. Download the Pascal Voc dataset (Can be downloaded from the
   [PASCAL VOC website](http://host.robots.ox.ac.uk/pascal/VOC/))

1. Create a CVAT task with the following labels:

   ```bash
   aeroplane bicycle bird boat bottle bus car cat chair cow diningtable
   dog horse motorbike person pottedplant sheep sofa train tvmonitor
   ```

   You can add `~checkbox=difficult:false ~checkbox=truncated:false`
   attributes for each label if you want to use them.

   Select interesting image files (See [Creating an annotation task](cvat/apps/documentation/user_guide.md#creating-an-annotation-task) guide for details)

1. zip the corresponding annotation files

1. click `Upload annotation` button, choose `Pascal VOC ZIP 1.1`

   and select the zip file with annotations from previous step.
   It may take some time.

### [YOLO](https://pjreddie.com/darknet/yolo/)<a id="yolo" />

- [Format specification](https://github.com/AlexeyAB/darknet#how-to-train-to-detect-your-custom-objects)
- supported annotations: Rectangles

#### YOLO export

Downloaded file: a zip archive with following structure:

```bash
archive.zip/
├── obj.data
├── obj.names
├── obj_<subset>_data
│   ├── image1.txt
│   └── image2.txt
└── train.txt # list of subset image paths

# the only valid subsets are: train, valid
# train.txt and valid.txt:
obj_<subset>_data/image1.jpg
obj_<subset>_data/image2.jpg

# obj.data:
classes = 3 # optional
names = obj.names
train = train.txt
valid = valid.txt # optional
backup = backup/ # optional

# obj.names:
cat
dog
airplane

# image_name.txt:
# label_id - id from obj.names
# cx, cy - relative coordinates of the bbox center
# rw, rh - relative size of the bbox
# label_id cx cy rw rh
1 0.3 0.8 0.1 0.3
2 0.7 0.2 0.3 0.1
```

Each annotation `*.txt` file has a name that corresponds to the name of
the image file (e. g. `frame_000001.txt` is the annotation
for the `frame_000001.jpg` image).
The `*.txt` file structure: each line describes label and bounding box
in the following format `label_id cx cy w h`.
`obj.names` contains the ordered list of label names.

#### YOLO import

Uploaded file: a zip archive of the same structure as above
It must be possible to match the CVAT frame (image name)
and annotation file name. There are 2 options:

1. full match between image name and name of annotation `*.txt` file
   (in cases when a task was created from images or archive of images).

1. match by frame number (if CVAT cannot match by name). File name
   should be in the following format `<number>.jpg` .
   It should be used when task was created from a video.

#### How to create a task from YOLO formatted dataset (from VOC for example)

1. Follow the official [guide](https://pjreddie.com/darknet/yolo/)(see Training YOLO on VOC section)
   and prepare the YOLO formatted annotation files.

1. Zip train images

```bash
zip images.zip -j -@ < train.txt
```

1. Create a CVAT task with the following labels:

   ```bash
   aeroplane bicycle bird boat bottle bus car cat chair cow diningtable dog
   horse motorbike person pottedplant sheep sofa train tvmonitor
   ```

   Select images. zip as data. Most likely you should use `share`
   functionality because size of images. zip is more than 500Mb.
   See [Creating an annotation task](cvat/apps/documentation/user_guide.md#creating-an-annotation-task)
   guide for details.

1. Create `obj.names` with the following content:

   ```bash
   aeroplane
   bicycle
   bird
   boat
   bottle
   bus
   car
   cat
   chair
   cow
   diningtable
   dog
   horse
   motorbike
   person
   pottedplant
   sheep
   sofa
   train
   tvmonitor
   ```

1. Zip all label files together (we need to add only label files that correspond to the train subset)

   ```bash
   cat train.txt | while read p; do echo ${p%/*/*}/labels/${${p##*/}%%.*}.txt; done | zip labels.zip -j -@ obj.names
   ```

1. Click `Upload annotation` button, choose `YOLO 1.1` and select the zip

   file with labels from the previous step.

### [MS COCO Object Detection](http://cocodataset.org/#format-data)<a id="coco" />

- [Format specification](https://github.com/openvinotoolkit/datumaro/blob/develop/docs/formats/coco_user_manual.md#format-specification)

#### COCO export

Downloaded file: a zip archive with the structure described [here](https://github.com/openvinotoolkit/datumaro/blob/develop/docs/formats/coco_user_manual.md#load-coco-dataset)

- supported annotations: Polygons, Rectangles
- supported attributes:
  - `is_crowd` (checkbox or integer with values 0 and 1) -
    specifies that the instance (an object group) should have an
    RLE-encoded mask in the `segmentation` field. All the grouped shapes
    are merged into a single mask, the largest one defines all
    the object properties
  - `score` (number) - the annotation `score` field
  - arbitrary attributes - will be stored in the `attributes` annotation section

Support for COCO tasks via Datumaro is described [here](https://github.com/openvinotoolkit/datumaro/blob/develop/docs/formats/coco_user_manual.md#export-to-coco)
For example, [support for COCO keypoints over Datumaro](https://github.com/openvinotoolkit/cvat/issues/2910#issuecomment-726077582):

1. Install [Datumaro](https://github.com/openvinotoolkit/datumaro)
  `pip install datumaro`
1. Export the task in the `Datumaro` format, unzip
1. Export the Datumaro project in `coco` / `coco_person_keypoints` formats
  `datum export -f coco -p path/to/project [-- --save-images]`

This way, one can export CVAT points as single keypoints or
keypoint lists (without the `visibility` COCO flag).

#### COCO import

Uploaded file: a single unpacked `*.json` or a zip archive with the [decribed structure](https://github.com/openvinotoolkit/datumaro/blob/develop/docs/formats/coco_user_manual.md#load-coco-dataset) (without images).

- supported annotations: Polygons, Rectangles (if the `segmentation` field is empty)

#### How to create a task from MS COCO dataset

1. Download the [MS COCO dataset](https://github.com/openvinotoolkit/datumaro/blob/develop/docs/formats/coco_user_manual.md#load-COCO-dataset).

   For example `val images` and `instances` annotations

1. Create a CVAT task with the following labels:

   ```bash
   person bicycle car motorcycle airplane bus train truck boat "traffic light" "fire hydrant" "stop sign" "parking meter" bench bird cat dog horse sheep cow elephant bear zebra giraffe backpack umbrella handbag tie suitcase frisbee skis snowboard "sports ball" kite "baseball bat" "baseball glove" skateboard surfboard "tennis racket" bottle "wine glass" cup fork knife spoon bowl banana apple sandwich orange broccoli carrot "hot dog" pizza donut cake chair couch "potted plant" bed "dining table" toilet tv laptop mouse remote keyboard "cell phone" microwave oven toaster sink refrigerator book clock vase scissors "teddy bear" "hair drier" toothbrush
   ```

1. Select `val2017.zip` as data
   (See [Creating an annotation task](cvat/apps/documentation/user_guide.md#creating-an-annotation-task)
   guide for details)

1. Unpack `annotations_trainval2017.zip`

1. click `Upload annotation` button,
   choose `COCO 1.1` and select `instances_val2017.json`
   annotation file. It can take some time.

### [TFRecord](https://www.tensorflow.org/tutorials/load_data/tf_records)<a id="tfrecord" />

TFRecord is a very flexible format, but we try to correspond the
format that used in
[TF object detection](https://github.com/tensorflow/models/tree/master/research/object_detection)
with minimal modifications.

Used feature description:

```python
image_feature_description = {
    'image/filename': tf.io.FixedLenFeature([], tf.string),
    'image/source_id': tf.io.FixedLenFeature([], tf.string),
    'image/height': tf.io.FixedLenFeature([], tf.int64),
    'image/width': tf.io.FixedLenFeature([], tf.int64),
    # Object boxes and classes.
    'image/object/bbox/xmin': tf.io.VarLenFeature(tf.float32),
    'image/object/bbox/xmax': tf.io.VarLenFeature(tf.float32),
    'image/object/bbox/ymin': tf.io.VarLenFeature(tf.float32),
    'image/object/bbox/ymax': tf.io.VarLenFeature(tf.float32),
    'image/object/class/label': tf.io.VarLenFeature(tf.int64),
    'image/object/class/text': tf.io.VarLenFeature(tf.string),
}
```

#### TFRecord export

Downloaded file: a zip archive with following structure:

```bash
taskname.zip/
├── default.tfrecord
└── label_map.pbtxt

# label_map.pbtxt
item {
	id: 1
	name: 'label_0'
}
item {
	id: 2
	name: 'label_1'
}
...
```

- supported annotations: Rectangles, Polygons (as masks, manually over [Datumaro](https://github.com/openvinotoolkit/datumaro/blob/develop/docs/user_manual.md))

How to export masks:
1. Export annotations in `Datumaro` format
1. Apply `polygons_to_masks` and `boxes_to_masks` transforms
  ```bash
  datum transform -t polygons_to_masks -p path/to/proj -o ptm
  datum transform -t boxes_to_masks -p ptm -o btm
  ```
1. Export in the `TF Detection API` format
  ```bash
  datum export -f tf_detection_api -p btm [-- --save-images]
  ```

#### TFRecord import

Uploaded file: a zip archive of following structure:

```bash
taskname.zip/
└── <any name>.tfrecord
```

- supported annotations: Rectangles

#### How to create a task from TFRecord dataset (from VOC2007 for example)

1. Create `label_map.pbtxt` file with the following content:

```js
item {
    id: 1
    name: 'aeroplane'
}
item {
    id: 2
    name: 'bicycle'
}
item {
    id: 3
    name: 'bird'
}
item {
    id: 4
    name: 'boat'
}
item {
    id: 5
    name: 'bottle'
}
item {
    id: 6
    name: 'bus'
}
item {
    id: 7
    name: 'car'
}
item {
    id: 8
    name: 'cat'
}
item {
    id: 9
    name: 'chair'
}
item {
    id: 10
    name: 'cow'
}
item {
    id: 11
    name: 'diningtable'
}
item {
    id: 12
    name: 'dog'
}
item {
    id: 13
    name: 'horse'
}
item {
    id: 14
    name: 'motorbike'
}
item {
    id: 15
    name: 'person'
}
item {
    id: 16
    name: 'pottedplant'
}
item {
    id: 17
    name: 'sheep'
}
item {
    id: 18
    name: 'sofa'
}
item {
    id: 19
    name: 'train'
}
item {
    id: 20
    name: 'tvmonitor'
}
```

1. Use [create_pascal_tf_record.py](https://github.com/tensorflow/models/blob/master/research/object_detection/dataset_tools/create_pascal_tf_record.py)

to convert VOC2007 dataset to TFRecord format.
As example:

```bash
python create_pascal_tf_record.py --data_dir <path to VOCdevkit> --set train --year VOC2007 --output_path pascal.tfrecord --label_map_path label_map.pbtxt
```

1. Zip train images

   ```bash
   cat <path to VOCdevkit>/VOC2007/ImageSets/Main/train.txt | while read p; do echo <path to VOCdevkit>/VOC2007/JPEGImages/${p}.jpg  ; done | zip images.zip -j -@
   ```

1. Create a CVAT task with the following labels:

   ```bash
   aeroplane bicycle bird boat bottle bus car cat chair cow diningtable dog horse motorbike person pottedplant sheep sofa train tvmonitor
   ```

   Select images. zip as data.
   See [Creating an annotation task](cvat/apps/documentation/user_guide.md#creating-an-annotation-task)
   guide for details.

1. Zip `pascal.tfrecord` and `label_map.pbtxt` files together

   ```bash
   zip anno.zip -j <path to pascal.tfrecord> <path to label_map.pbtxt>
   ```

1. Click `Upload annotation` button, choose `TFRecord 1.0` and select the zip file

   with labels from the previous step. It may take some time.

### [MOT sequence](https://arxiv.org/pdf/1906.04567.pdf)<a id="mot" />

#### MOT export

Downloaded file: a zip archive of the following structure:

```bash
taskname.zip/
├── img1/
|   ├── image1.jpg
|   └── image2.jpg
└── gt/
    ├── labels.txt
    └── gt.txt

# labels.txt
cat
dog
person
...

# gt.txt
# frame_id, track_id, x, y, w, h, "not ignored", class_id, visibility, <skipped>
1,1,1363,569,103,241,1,1,0.86014
...

```

- supported annotations: Rectangle shapes and tracks
- supported attributes: `visibility` (number), `ignored` (checkbox)

#### MOT import

Uploaded file: a zip archive of the structure above or:

```bash
taskname.zip/
├── labels.txt # optional, mandatory for non-official labels
└── gt.txt
```

- supported annotations: Rectangle tracks

### [MOTS PNG](https://www.vision.rwth-aachen.de/page/mots)<a id="mots" />

#### MOTS PNG export

Downloaded file: a zip archive of the following structure:

```bash
taskname.zip/
└── <any_subset_name>/
    |   images/
    |   ├── image1.jpg
    |   └── image2.jpg
    └── instances/
        ├── labels.txt
        ├── image1.png
        └── image2.png

# labels.txt
cat
dog
person
...
```

- supported annotations: Rectangle and Polygon tracks

#### MOTS PNG import

Uploaded file: a zip archive of the structure above

- supported annotations: Polygon tracks

### [LabelMe](http://labelme.csail.mit.edu/Release3.0)<a id="labelme" />

#### LabelMe export

Downloaded file: a zip archive of the following structure:

```bash
taskname.zip/
├── img1.jpg
└── img1.xml
```

- supported annotations: Rectangles, Polygons (with attributes)

#### LabelMe import

Uploaded file: a zip archive of the following structure:

```bash
taskname.zip/
├── Masks/
|   ├── img1_mask1.png
|   └── img1_mask2.png
├── img1.xml
├── img2.xml
└── img3.xml
```

- supported annotations: Rectangles, Polygons, Masks (as polygons)

### [ImageNet](http://www.image-net.org)<a id="imagenet" />

#### ImageNet export

Downloaded file: a zip archive of the following structure:

```bash
# if we save images:
taskname.zip/
├── label1/
|   ├── label1_image1.jpg
|   └── label1_image2.jpg
└── label2/
    ├── label2_image1.jpg
    ├── label2_image3.jpg
    └── label2_image4.jpg

# if we keep only annotation:
taskname.zip/
├── <any_subset_name>.txt
└── synsets.txt

```

- supported annotations: Labels

#### ImageNet import

Uploaded file: a zip archive of the structure above

- supported annotations: Labels

### [CamVid](http://mi.eng.cam.ac.uk/research/projects/VideoRec/CamVid/)<a id="camvid" />

#### CamVid export

Downloaded file: a zip archive of the following structure:

```bash
taskname.zip/
├── labelmap.txt # optional, required for non-CamVid labels
├── <any_subset_name>/
|   ├── image1.png
|   └── image2.png
├── <any_subset_name>annot/
|   ├── image1.png
|   └── image2.png
└── <any_subset_name>.txt

# labelmap.txt
# color (RGB) label
0 0 0 Void
64 128 64 Animal
192 0 128 Archway
0 128 192 Bicyclist
0 128 64 Bridge
```

Mask is a `png` image with 1 or 3 channels where each pixel
has own color which corresponds to a label.
`(0, 0, 0)` is used for background by default.

- supported annotations: Rectangles, Polygons

#### CamVid import

Uploaded file: a zip archive of the structure above

- supported annotations: Polygons

### [WIDER Face](http://shuoyang1213.me/WIDERFACE/)<a id="widerface" />

#### WIDER Face export

Downloaded file: a zip archive of the following structure:

```bash
taskname.zip/
├── labels.txt # optional
├── wider_face_split/
│   └── wider_face_<any_subset_name>_bbx_gt.txt
└── WIDER_<any_subset_name>/
    └── images/
        ├── 0--label0/
        │   └── 0_label0_image1.jpg
        └── 1--label1/
            └── 1_label1_image2.jpg
```

- supported annotations: Rectangles (with attributes), Labels
- supported attributes:
  - `blur`, `expression`, `illumination`, `pose`, `invalid`
  - `occluded` (both the annotation property & an attribute)

#### WIDER Face import

Uploaded file: a zip archive of the structure above

- supported annotations: Rectangles (with attributes), Labels
- supported attributes:
  - `blur`, `expression`, `illumination`, `occluded`, `pose`, `invalid`

### [VGGFace2](https://github.com/ox-vgg/vgg_face2)<a id="vggface2" />

#### VGGFace2 export

Downloaded file: a zip archive of the following structure:

```bash
taskname.zip/
├── labels.txt # optional
├── <any_subset_name>/
|   ├── label0/
|   |   └── image1.jpg
|   └── label1/
|       └── image2.jpg
└── bb_landmark/
    ├── loose_bb_<any_subset_name>.csv
    └── loose_landmark_<any_subset_name>.csv
# labels.txt
# n000001 car
label0 <class0>
label1 <class1>
```

- supported annotations: Rectangles, Points (landmarks - groups of 5 points)

#### VGGFace2 import

Uploaded file: a zip archive of the structure above

- supported annotations: Rectangles, Points (landmarks - groups of 5 points)

### [Market-1501](https://www.aitribune.com/dataset/2018051063)<a id="market1501" />

#### Market-1501 export

Downloaded file: a zip archive of the following structure:

```bash
taskname.zip/
├── bounding_box_<any_subset_name>/
│   └── image_name_1.jpg
└── query
    ├── image_name_2.jpg
    └── image_name_3.jpg
# if we keep only annotation:
taskname.zip/
└── images_<any_subset_name>.txt
# images_<any_subset_name>.txt
query/image_name_1.jpg
bounding_box_<any_subset_name>/image_name_2.jpg
bounding_box_<any_subset_name>/image_name_3.jpg
# image_name = 0001_c1s1_000015_00.jpg
0001 - person id
c1 - camera id (there are totally 6 cameras)
s1 - sequence
000015 - frame number in sequence
00 - means that this bounding box is the first one among the several
```

- supported annotations: Label `market-1501` with atrributes (`query`, `person_id`, `camera_id`)

#### Market-1501 import

Uploaded file: a zip archive of the structure above

- supported annotations: Label `market-1501` with atrributes (`query`, `person_id`, `camera_id`)

### [ICDAR13/15](https://rrc.cvc.uab.es/?ch=2)<a id="icdar" />

#### ICDAR13/15 export

Downloaded file: a zip archive of the following structure:

```bash
# word recognition task
taskname.zip/
└── word_recognition/
    └── <any_subset_name>/
        ├── images
        |   ├── word1.png
        |   └── word2.png
        └── gt.txt
# text localization task
taskname.zip/
└── text_localization/
    └── <any_subset_name>/
        ├── images
        |   ├── img_1.png
        |   └── img_2.png
        ├── gt_img_1.txt
        └── gt_img_1.txt
#text segmentation task
taskname.zip/
└── text_localization/
    └── <any_subset_name>/
        ├── images
        |   ├── 1.png
        |   └── 2.png
        ├── 1_GT.bmp
        ├── 1_GT.txt
        ├── 2_GT.bmp
        └── 2_GT.txt
```

**Word recognition task**:

- supported annotations: Label `icdar` with attribute `caption`

**Text localization task**:

- supported annotations: Rectangles and Polygons with label `icdar`
  and attribute `text`

**Text segmentation task**:

- supported annotations: Rectangles and Polygons with label `icdar`
  and attributes `index`, `text`, `color`, `center`

#### ICDAR13/15 import

Uploaded file: a zip archive of the structure above

**Word recognition task**:

- supported annotations: Label `icdar` with attribute `caption`

**Text localization task**:

- supported annotations: Rectangles and Polygons with label `icdar`
  and attribute `text`

**Text segmentation task**:

- supported annotations: Rectangles and Polygons with label `icdar`
  and attributes `index`, `text`, `color`, `center`
