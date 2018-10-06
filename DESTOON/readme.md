# DESTOON B2B网站管理系统 前台getshell漏洞

------

#### 前言

​	之前好像还没碰到过这种多文件上传的思路。

#### 分析过程

​	漏洞的位置在 module/member/avatar.inc.php 这个页面，所以直接进去先看看，直接看上传的部分。

```php
case 'upload':
		if(!$_FILES['file']['size']) {
			if($DT_PC) dheader('?action=html&reload='.$DT_TIME);
			exit('{"error":1,"message":"Error FILE"}');
		}
		require DT_ROOT.'/include/upload.class.php';
		$ext = file_ext($_FILES['file']['name']);	//获取文件的后缀名
		$name = 'avatar'.$_userid.'.'.$ext;		//构造文件名
		$file = DT_ROOT.'/file/temp/'.$name;	//构造文件路径
		if(is_file($file)) file_del($file);
		$upload = new upload($_FILES, 'file/temp/', $name, 'jpg|jpeg|gif|png');
		$upload->adduserid = false;
		if($upload->save()) {
			$file = DT_ROOT.'/file/temp/'.$name;
         ...
```

​	文件上传 -> 文件保存 -> 对文件格式进行处理		继续跟进里面 new 的 upload 这个类。

```php
class upload {
    var $file;
    var $file_name;
    var $file_size;
    var $file_type;
	var $file_error;
    var $savename;
    var $savepath;
	var $saveto;
    var $fileformat = '';
    var $overwrite = false;
    var $maxsize;
    var $ext;
    var $errmsg = errmsg;
	var $userid;
	var $image;
	var $adduserid = true;

    function __construct($_file, $savepath, $savename = '', $fileformat = '') {
		global $DT, $_userid;
		foreach($_file as $file) {	//多文件上传可绕过
			$this->file = $file['tmp_name'];
			$this->file_name = $file['name'];
			$this->file_size = $file['size'];
			$this->file_type = $file['type'];
			$this->file_error = $file['error'];
		}
		$this->userid = $_userid;
		$this->ext = file_ext($this->file_name);
		$this->fileformat = $fileformat ? $fileformat : $DT['uploadtype'];
		$this->maxsize = $DT['uploadsize'] ? $DT['uploadsize']*1024 : 2048*1024;
		$this->savepath = $savepath;
		$this->savename = $savename;
    }

    function upload($_file, $savepath, $savename = '', $fileformat = '') {
		$this->__construct($_file, $savepath, $savename, $fileformat);
    }

	function save() {
		include load('include.lang');
        if($this->file_error) return $this->_('Error(21)'.$L['upload_failed'].' ('.$L['upload_error_'.$this->file_error].')');
		if($this->maxsize > 0 && $this->file_size > $this->maxsize) return $this->_('Error(22)'.$L['upload_size_limit'].' ('.intval($this->maxsize/1024).'Kb)');
        if(!$this->is_allow()) return $this->_('Error(23)'.$L['upload_not_allow']);
        $this->set_savepath($this->savepath);
        $this->set_savename($this->savename);
        if(!is_writable(DT_ROOT.'/'.$this->savepath)) return $this->_('Error(24)'.$L['upload_unwritable']);
		if(!is_uploaded_file($this->file)) return $this->_('Error(25)'.$L['upload_failed']);
		if(!move_uploaded_file($this->file, DT_ROOT.'/'.$this->saveto)) return $this->_('Error(26)'.$L['upload_failed']);
		$this->image = $this->is_image();
		if(DT_CHMOD) @chmod(DT_ROOT.'/'.$this->saveto, DT_CHMOD);
        return true;
	}

    function is_allow() {
		if(!$this->fileformat) return false;
		if(!preg_match("/^(".$this->fileformat.")$/i", $this->ext)) return false;
		if(preg_match("/^(php|phtml|php3|php4|jsp|exe|dll|cer|shtml|shtm|asp|asa|aspx|asax|ashx|cgi|fcgi|pl)$/i", $this->ext)) return false;
		return true;
    }
    function set_savename($savename) {
        if($savename) {
            $this->savename = $this->adduserid ? str_replace('.'.$this->ext, $this->userid.'.'.$this->ext, $savename) : $savename;
        } else {
            $name = date('His', DT_TIME).mt_rand(10, 99);
            $this->savename = $this->adduserid ? $name.$this->userid.'.'.$this->ext : $name.'.'.$this->ext;
        }
		$this->saveto = $this->savepath.$this->savename;		
        if(!$this->overwrite && is_file(DT_ROOT.'/'.$this->saveto)) {
			$i = 1;
			while($i) {
				$saveto = str_replace('.'.$this->ext, '('.$i.').'.$this->ext, $this->saveto);
				if(is_file(DT_ROOT.'/'.$saveto)) {
					$i++;
					continue; 
				} else {
					$this->saveto = $saveto; 
					break;
				}
			}
        }
    }
```

​	在这个类中的 __construct 部分，用了一个 foreach 来获取文件的信息，但这里传入的是 $_FILE 就是整个 file变量，所以如果我们传入多个文件的话，在遍历 $_FILE 变量的时候，就会存在一个变量覆盖的问题。传入两个文件的时候，代码获取到的是我们传入的第二个文件的信息，检验的当然也是我们第二个文件的信息。

​	然后再看到 upload 中的 save 这个函数，利用传入多文件可以绕过验证，但重点是要使文件按我们想要的格式保存下来。直接看 move_uploaded_file 这个函数，路径参数为 DT_ROOT.'/'.$this->saveto，所以继续跟进 $this->saveto 这个变量，发现它在 set_savename 函数中被赋值  $this->saveto = $this->savepath.$this->savename;	  ，恰好就是前面传入的，也就是我们可以控制传入的文件名（为传入的第一个文件的名字  xxx.xxx）。

​	当然，代码后面又对这个文件再进行了一次检验，不过这时文件已经保存下来了。得用条件竞争去读到文件。

------

#### 单文件上传

```php+HTML
<html>
<head>
<meta charset="utf-8">
<title>菜鸟教程(runoob.com)</title>
</head>
<body>
<form action="test.php" method="post" enctype="multipart/form-data">
//用form上传文件时，一定要加上属性内容 enctype="multipart/form-data"，否则用$_FILES[filename]获取文件信息时会报异常。
    <label for="file">文件名：</label>
    <input type="file" name="file" id="file"><br>
    <input type="submit" name="submit" value="提交">
</form>
</body>
</html>

<?php
var_dump($_FILES);
?>

array(1) {
  ["file"]=>        //表单中的name
  array(5) {
    ["name"]=>        //文件本身的名字，对应包里的filename
    string(5) "s.php"
    ["type"]=>            //文件
    string(24) "application/octet-stream"        //文件的mime类型，由Content-Type控制
    ["tmp_name"]=>
    string(22) "C:\Windows\php31E6.tmp"
    ["error"]=>
    int(0)
    ["size"]=>
    int(21)
  }
}
```



#### 多文件上传

```php+HTML
<form action="test.php" method="post" enctype="multipart/form-data">
        请选择我的上传文件
        <input type="file" name="myfil1"/>
        <input type="file" name="myfil2"/>
        <input type="file" name="myfil3"/>
        <input type="submit" value="上传" />
    </form>
   
<?php
var_dump($_FILE);
?>
/*
    array(3) {
  ["myfil1"]=>
  array(5) {
    ["name"]=>
    string(5) "s.php"
    ["type"]=>
    string(24) "application/octet-stream"
    ["tmp_name"]=>
    string(22) "C:\Windows\phpB825.tmp"
    ["error"]=>
    int(0)
    ["size"]=>
    int(21)
  }
  ["myfil2"]=>
  array(5) {
    ["name"]=>
    string(5) "s.php"
    ["type"]=>
    string(24) "application/octet-stream"
    ["tmp_name"]=>
    string(22) "C:\Windows\phpB836.tmp"
    ["error"]=>
    int(0)
    ["size"]=>
    int(21)
  }
  ["myfil3"]=>
  array(5) {
    ["name"]=>
    string(5) "s.php"
    ["type"]=>
    string(24) "application/octet-stream"
    ["tmp_name"]=>
    string(22) "C:\Windows\phpB837.tmp"
    ["error"]=>
    int(0)
    ["size"]=>
    int(21)
  }
}
*/
```

