## yccms3.3 logo处的文件上传分析（一个水洞）

------

#### 前言

​	因为想找一些cms来学学代码审计，但又不想去整个cms那样翻，于是在seebug上随意找了一些洞来自己分析。

#### 分析过程

​	在seebug上看到这个洞的标题描述是文件上传，但并没有关于这个洞的其他详细信息。于是把源码下载下来后，把站搭了起来，并在源码里全局的搜索了一下upload的功能，定位到了这几个文件。

CallAction.class.php	FileUpload.class.php	LogoUpload.class.php

​	在CallAction.class.php中有这么两个文件上传的函数，分别负责logo和编辑器的图片上传。

```php
public function upLoad() {
		if (isset($_POST['send'])) {
			$_logoupload = new LogoUpload('pic',$_POST['MAX_FILE_SIZE']);//新构造一个上传的类
			//exit();//test
			$_path = $_logoupload->getPath();
			$_img = new Image($_path);
			$_img->xhImg(960,0);
			$_img->out();
			//echo $_path;
			$_logoupload->alertOpenerClose('图片上传成功！','..'.$_path);
		} else {
			exit('警告：文件过大或者其他未知错误导致浏览器崩溃！');
		}
	}
	//xheditor编辑器专用上传
	public function xhUp() {
		if (isset($_GET['type'])) {
			$_fileupload = new FileUpload('filedata',10);
			$_err=$_fileupload->checkError();
			$_path = $_fileupload->getPath();
			$_msg="'..$_path'";
			$_img = new Image($_path);
			$_img->xhImg(650,0);
			$_img->out();
			echo "{'err':'".$_err."','msg':".$_msg."}";
			exit();
		} else {
		Tool::alertBack('警告：由于非法操作导致上传失败！');
		}
	}
```

​	两段个函数的代码其实差不多，都是先new一个类进行上传操作，然后再对图片进行操作。所以我们先跟进它new的类（同样，两个上传类的代码也是差不多的，所以只看一个就够了

​	LogoUpload.class.php

```php
class LogoUpload {
	private $error;			//错误代码
	private $maxsize;		//表单最大值
	private $type;				//类型
	private $typeArr = array('image/png','image/x-png');		//类型合集
	private $path;				//目录路径
	private $name;			//文件名
	private $tmp;				//临时文件
	private $linkpath;		//链接路径
	
	//构造方法，初始化
	public function __construct($_file,$_maxsize) {//文件名记为pic
		$this->error = $_FILES[$_file]['error'];//记录文件信息
		$this->maxsize = $_maxsize / 1024;
		$this->type = $_FILES[$_file]['type'];//可以控制
		$this->path = ROOT_PATH.'/'.UPLOGO;
		$this->name = $_FILES[$_file]['name'];
		$this->tmp = $_FILES[$_file]['tmp_name'];
		$this->checkError();
		$this->checkType();
		$this->checkPath();
		$this->moveUpload();//上传
	}
    ...
//验证类型
	private function checkType() {
		if (!in_array($this->type,$this->typeArr)) {
			Tool::alertBack('警告：LOGO图片必须是PNG格式！');
		}
	}
```

​	可以看到，在__construct中，代码先读入文件的一些信息，然后经过几个没有意义的检查就上传了。。。代码只检查了上传文件的 content-type ，而这个是我们可以直接抓包修改的。所以可以上传一个任意文件，然后修改 Content-Type 为 image/png 就可以直接上传了（/YCCMS/view/index/images/logo.php），实际上传的时候，因为在后面有一段代码在修改图片的尺寸，所以会报错，但文件实际已经传了上去了（就算他后面再删掉文件也能用条件竞争读到

​	在logo处和编辑器处都有这个问题（大水洞一个