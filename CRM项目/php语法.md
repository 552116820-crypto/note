```php
• 可以，PHP 里这 3 个符号分别是不同层级的访问方式：

  1. $：变量
  表示一个变量。

  $name = "Tom";
  $age = 18;

  也用于对象变量：

  $user = new User();

  所以 $user 的意思是：变量 user。

  2. ->：访问对象的属性或方法
  当一个变量里装的是“对象”时，用 -> 访问它的成员。

  $user->name = "Tom";
  echo $user->name;
  $user->login();

  意思是：

  - $user->name：访问对象的属性 name
  - $user->login()：调用对象的方法 login()

  3. ::：访问类的静态成员、常量，或指定类作用域
  这是“类级别”的访问，不依赖某个具体对象。

  class User {
      public static $count = 0;
      const TYPE = "admin";

      public static function hello() {
          echo "hi";
      }
  }

  echo User::$count;
  echo User::TYPE;
  User::hello();

  意思是：

  - User::$count：访问类的静态属性
  - User::TYPE：访问类常量
  - User::hello()：调用静态方法

  对比着看：

  $user = new User();

  $user->name;     // 对象属性
  $user->login();  // 对象方法

  User::$count;    // 类的静态属性
  User::hello();   // 类的静态方法
  User::TYPE;      // 类常量

  一句话区别：

  - $：这是个变量
  - ->：从对象里取东西
  - ::：从类本身里取东西

  类比 Python：

  - $user->name 类似 Python 的 user.name
  - User::hello() 类似 Python 的 User.hello()
  - $ 在 Python 里没有对应符号，Python 变量直接写 user


```