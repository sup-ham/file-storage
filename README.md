<p align="center">
    <a href="https://github.com/yii2tech" target="_blank">
        <img src="https://avatars2.githubusercontent.com/u/12951949" height="100px">
    </a>
    <h1 align="center">Lite version of package yii2tech/file-storage the file storage abstraction layer for Yii2</h1>
    <br>
</p>

This extension provides file storage abstraction layer for Yii2.

For license information check the [LICENSE](LICENSE.md)-file.

[![Latest Stable Version](https://poser.pugx.org/yii2tech/file-storage/v/stable.png)](https://packagist.org/packages/yii2tech/file-storage)
[![Total Downloads](https://poser.pugx.org/yii2tech/file-storage/downloads.png)](https://packagist.org/packages/yii2tech/file-storage)
[![Build Status](https://travis-ci.org/yii2tech/file-storage.svg?branch=master)](https://travis-ci.org/yii2tech/file-storage)


Installation
------------

The preferred way to install this extension is through [composer](http://getcomposer.org/download/).

Either run

```
php composer.phar config repositories.fslite vcs https://github.com/sup-ham/yii2-filestorage.git
php composer.phar require --prefer-dist yii2tech/file-storage
```

or add

```json
"yii2tech/file-storage": "*"
```

to the require section and add new repo url

```
  "repositories" : [
    {
      "type": "vcs",
      "url": "https://github.com/sup-ham/yii2-filestorage.git"
    }
  ]
```
to your composer.json file


Usage
-----

This extension provides file storage abstraction layer for Yii2.
This abstraction introduces 2 main terms: 'storage' and 'bucket'.
'Storage' - is a unit, which is able to store files using some particular approach.
'Bucket' - is a logical part of the storage, which has own specific attributes and serves some logical mean.
Each time you need to read/write a file you should do it via particular bucket, which is always belongs to the
file storage.

Example application configuration:

```php
return [
    'components' => [
        'fileStorage' => [
            'class' => 'yii2tech\filestorage\local\Storage',
            'basePath' => '@webroot/files',
            'baseUrl' => '@web/files',
            'dirPermission' => 0775,
            'filePermission' => 0755,
            'buckets' => [
                'tempFiles' => [
                    'baseSubPath' => 'temp',
                    'fileSubDirTemplate' => '{^name}/{^^name}',
                ],
                'imageFiles' => [
                    'baseSubPath' => 'image',
                    'fileSubDirTemplate' => '{ext}/{^name}/{^^name}',
                ],
            ]
        ],
        // ...
    ],
    // ...
];
```

Example usage:

```php
$bucket = Yii::$app->fileStorage->getBucket('tempFiles');

$bucket->saveFileContent('foo.txt', 'Foo content'); // create file with content
$bucket->deleteFile('foo.txt'); // deletes file from bucket
$bucket->copyFileIn('/path/to/source/file.txt', 'file.txt'); // copy file into the bucket
$bucket->copyFileOut('file.txt', '/path/to/destination/file.txt'); // copy file from the bucket
var_dump($bucket->fileExists('file.txt')); // outputs `true`
echo $bucket->getFileUrl('file.txt'); // outputs: 'http://domain.com/files/f/i/file.txt'
```

Following file storages are available with this extension:
 - [[\yii2tech\filestorage\local\Storage]] - stores files on the OS local file system.
 - [[\yii2tech\filestorage\hub\Storage]] - allows combination of different file storages.

Please refer to the particular storage class for more details.

**Heads up!** Some of the storages may require additional libraries or PHP extensions, which are not
required with this package by default, to be installed. Please check particular storage class documentation
for the details.


## Accessing files by URL <span id="accessing-files-by-url"></span>

In order to get URL leading to the stored file, you should use [[\yii2tech\filestorage\BucketInterface::getFileUrl()]] method:

```php
$bucket = Yii::$app->fileStorage->getBucket('tempFiles');

$fileUrl = $bucket->getFileUrl('image.jpg');
```

In case particular storage does not provide native URL file access, or it is not available or not desirable by some reason,
you can setup composition of the file URL via Yii URL route mechanism. You need to setup `baseUrl` to be an array, containing
route, which leads to the Yii controller action, which will return the file content. For example:

```php
return [
    'components' => [
        'fileStorage' => [
            'class' => 'yii2tech\filestorage\local\Storage',
            'baseUrl' => ['/file/download'],
            // ...
        ],
        // ...
    ],
    // ...
];
```

With this configuration `getFileUrl()` method will use current application URL manager to create URL. Doing so it
will add bucket name as `bucket` parameter and file name as `filename` parameter. For example:

```php
use yii\helpers\Url;

$bucket = Yii::$app->fileStorage->getBucket('images');

$fileUrl = $bucket->getFileUrl('logo.png');
$manualUrl = Url::to(['/file/download', 'bucket' => 'images', 'filename' => 'logo.png']);
var_dump($fileUrl === $manualUrl); // outputs `true`
```

You may setup [[\yii2tech\filestorage\DownloadAction]] to handle file content web access. For example:

```php
class FileController extends \yii\web\Controller
{
    public function actions()
    {
        return [
            'download' => [
                'class' => 'yii2tech\filestorage\DownloadAction',
            ],
        ];
    }
}
```

> Tip: usage of the controller action for the file web access usually slower then native mechanism provided by
  file storage, however, you may put some extra logic into it, like allowing file access for logged in users only.


## Processing of the large files <span id="processing-of-the-large-files"></span>

Saving or reading large files, like > 100 MB, using such methods like [[\yii2tech\filestorage\BucketInterface::saveFileContent()]] or
[[\yii2tech\filestorage\BucketInterface::getFileContent()]], may easily exceed PHP memory limit, breaking the script.
You should use [[\yii2tech\filestorage\BucketInterface::openFile()]] method to create a file resource similar to the one
created via [[fopen()]] PHP function. Such resource can be read or written by blocks, keeping memory usage low.
For example:

```php
$bucket = Yii::$app->fileStorage->getBucket('tempFiles');

$resource = $bucket->openFile('new_file.dat', 'w');
fwrite($resource, 'content part1');
fwrite($resource, 'content part2');
// ...
fclose($resource);

$resource = $bucket->openFile('existing_file.dat', 'r');
while (!feof($resource)) {
    echo fread($resource, 1024);
}
fclose($resource);
```

> Note: You should prefer usage of simple modes like `r` and `w`, avoiding complex ones like `w+`, as they
  may be not supported by some storages.


## Logging <span id="logging"></span>

Each file operation performed by file storage component is logged.
In order to setup a log target, which can capture all entries related to file storage, you should
use category `yii2tech\filestorage\*`. For example:

```php
return [
    // ...
    'components' => [
        // ...
        'log' => [
            // ...
            'targets' => [
                [
                    'class' => 'yii\log\FileTarget',
                    'logFile' => '@runtime/logs/file-storage.log',
                    'categories' => ['yii2tech\filestorage\*'],
                ],
                // ...
            ],
        ],
    ],
];
```
