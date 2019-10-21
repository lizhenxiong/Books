# 重构 #

## 一，重构的原则

#### 1，重构的定义
   * （名词形式）对软件内部结构的一种调整，目的是在不改变软件可察行为的前提下，提高可理解性，降低修改成本。
   * （动词形式）使用一些列重构手法，在不改变软件可观察行为的前提下，调整其结构。

#### 2，软件开发的两顶帽子
   * 添加新功能时，不应该修改既有代码，只管添加新功能并通过测试。
   * 重构时不再添加新功能，只管改进程序结构，并通过已有测试。

#### 3，为何重构
   * 重构改进软件设计（Design）
   * 重构使软件更容易理解（Maintain）
   * 重构帮助找到BUG（Debug）
   * 重构提高编程速度（Efficiency）

#### 4，何时重构
   * 三次法则：事不过三，三则重构
   * 添加功能时重构（New Feature）
   * 修补错误时重构（Bug Fix）
   * 复审代码时重构（Code Review）

#### 5，何时不该重构
   * 既有代码太混乱，且不能正常工作，需要重写而不是重构。
   * 项目接近最后期限时，应该避免重构。	

#### 6，重构的目标
|为什么程序如此难以修改？|设计与重构的目标|
|:-|:-:|
|难以阅读的程序，难以修改|容易阅读|
|逻辑重复的程序，难以修改|所有逻辑都只在唯一地点指定|
|添加新行为时需要修改已有代码的程序，难以修改|新的改动不会危及现有行为|
|带复杂条件逻辑的程序，难以修改|尽可能简单表达条件逻辑|

#### 7，
   * 间接层的价值
      * 允许逻辑共享（避免重复代码）
      * 分开解释意图和实现（方法越短小，越容易起好名字揭示意图，单一职责）
      * 隔离变化（软件需要根据需求的变化不断修改，隔离缩小修改的范围）
      * 封装条件逻辑（多态）
   * 大多数重构都为程序引入了更多的间接层
      * 过多的间接层会导致代码的层次太深
      * 使代码难以阅读.因此要权衡加入间接层的利弊

## 二，重新组织函数

#### 1，Extract Method 提炼函数
> 将一段代码放进一个独立函数中，并让函数名称解释该函数的用途。增加可读性，函数粒度小更容易被复用和覆写。
```
修改前：
   $course = $this->getCourse($id);
   $price = $originPrice;
   $coinPrice = $course['originCoinPrice'];
   $courseSet = $this->getCourseSetService()->getCourseSet($course['courseSetId']);

   if (!empty($courseSet['discountId'])) {
      $price = $price * $courseSet['discount'] / 10;
      $coinPrice = $coinPrice * $courseSet['discount'] / 10;
   }

   list($fields['price'], $fields['coinPrice']) = array($price, $coinPrice);

修改后：
list($fields['price'], $fields['coinPrice']) = $this->calculateCoursePrice($course['id'], $fields['originPrice']);

protected function calculateCoursePrice($id, $originPrice)
{
   $course = $this->getCourse($id);
   $price = $originPrice;
   $coinPrice = $course['originCoinPrice'];
   $courseSet = $this->getCourseSetService()->getCourseSet($course['courseSetId']);

   if (!empty($courseSet['discountId'])) {
      $price = $price * $courseSet['discount'] / 10;
      $coinPrice = $coinPrice * $courseSet['discount'] / 10;
   }

   return array($price, $coinPrice);
}

```

#### 2，Inline Method 内联函数
> 在函数调用点插入函数本体，然后移除该函数。函数的本体与名称同样清楚易懂，间接层太多反而不易理解。
```
修改前：
调用函数
$recommendCount = $this->countWithJoinCourseSet($conditions);

独立函数，函数名和基本也方法同名，可读性一般，可以这么做，但不推荐
public function countWithJoinCourseSet($conditions)
{
   $conditions = $this->_prepareCourseConditions($conditions);

   return $this->getCourseDao()->countWithJoinCourseSet($conditions);
}


修改后：
$conditions = $this->_prepareCourseConditions($conditions);
在函数调用点直接使用函数本体
$recommendCount = $this->getCourseDao()->countWithJoinCourseSet($conditions);
```
#### 3，Inline Temp 内联临时变量
> 将所有对该变量的引用动作，替换为对它赋值的那个表达式自身。
```
出现此种情况是，大部分都可以用(Replace Temp with Query 以查询取代临时变量)来解决，但是若表达式只使用一次且表达式含义简单易懂，则不需要提炼表达式，则直接使用表达式赋值。例如：(三元表达式)。
```
#### 4，Replace Temp with Query 以查询取代临时变量
> 将一个表达式提炼到一个独立函数中，并将临时变量的引用点替换为对函数的调用。临时变量扩展为查询函数，就可以将使用范围扩展到整个类。减少临时变量，使函数更短更易维护。
```
修改前：
eg1:
$user['salt'] = base_convert(sha1(uniqid(mt_rand(), true)), 16, 36);

eg2;
$salt = base_convert(sha1(uniqid(mt_rand(), true)), 16, 36);

$fields = array(
   'salt' => $salt,
   'password' => $this->getPasswordEncoder()->encodePassword($password, $salt),
);

修改后：
$user['salt'] = $this->generateSalt();

$fields = array(
   'salt' => $this->generateSalt(),
   'password' => $this->getPasswordEncoder()->encodePassword($password, $salt),
);

protected function generateSalt()
{
   return base_convert(sha1(uniqid(mt_rand(), true)), 16, 36);
}
```

#### 5，Introduce Explaining Variable（引入解释性变量）
> 将该复杂表达式的结果放进一个临时变量，以变量名来解释其用途。
```
eg:
$course = !is_array($course) ? $this->getCourse(intval($course)) : $course;
```

#### 6，Split Temporary Variable（分解临时变量）
> 针对每次赋值，创造一个独立、对应的临时变量。临时变量会被多次赋值，容易产生理解歧义。如果变量被多次赋值（除了“循环变量”和“结果收集变量”），说明承担了多个职责，应该分解。
```
修改前：
result 变量两次赋值意义不一，产生歧义
if ('generateToken' == $params['apiUrl']) {
   $apiUserId = empty($params['apiUserId']) ? null : $params['apiUserId'];

   if (!empty($apiUserId)) {
         $user = $this->getUserService()->getUser($apiUserId);
         if (empty($user)) {
            $this->createNewException(UserException::NOTFOUND_USER());
         }
         $token = $this->getUserService()->makeToken(
            MobileBaseController::TOKEN_TYPE,
            $user['id'],
            time() + 3600 * 24 * 30
         );
   }
   $result = array('X-Auth-Token' => $token);
} else {
   $result = $this->sendApiVersion3($params);
}

修改后：

```

#### 7，Replace Method with Method object 函数对象取代函数
> 一个大型函数如果包含了很多临时变量，用Extract Method很难拆解，可以把函数放到一个新创建的类中，把临时变量变成类的实体变量，再用Extract Method拆解。
```
函数工具类：
eg: ArrayToolkit，DateToolkit 等等
```

## 三，在对象之间搬移特性

#### 1，Move Method 移动函数
> 类的行为做到单一职责，不要越俎代庖。如果一个类有太多行为，或一个类与另一个类有太多合作而形成高度耦合，就需要搬移函数。观察调用它的那一端、它调用的那一端，以及继承体系中它的任何一个重定义函数。根据“这个函数与哪个对象的交流比较多”，决定其移动路径。

#### 2，Move Field 搬移字段
> 如果一个类的字段在另一个类中使用更频繁，就考虑搬移它。

#### 3，Extract Class 提炼类
> 一个类应该是一个清楚地抽象，处理一些明确的职责。

#### 4，Inline Class 将类内联化
> nline Class （将类内联化）正好于Extract Class （提炼类）相反。如果一个类不再承担足够责任、不再有单独存在的理由。将这个类的所有特性搬移到另一个类中，然后移除原类。

## 四，重新组织数据

#### 1，Self Encapsulate Field自封装字段
> 为这个字段建立getter/setter函数，并且只以这些函数访问字段。直接访问一个字段，导致出现强耦合关系。直接访问的好处是易阅读。间接访问的好处是好管理，子类好覆写。
```
直接访问和间接访问都可以使用，根据自己的需求选择，ES中直接访问使用较多
eg:
直接访问
private $cachePath;

private $activitiesConfig;

public function __construct($cacheDir, $activitiesRootDir, $isDebug)
{
   $this->cachePath = implode(DIRECTORY_SEPARATOR, array($cacheDir, 'activities.php'));
   $activitiesConfig = new ConfigCache($this->cachePath, $isDebug);

   if (!$activitiesConfig->isFresh()) {
      $this->reGenerate($activitiesConfig, $activitiesRootDir);
   }

   $this->activitiesConfig = require $this->cachePath;
}

间接访问
private $activityConfig;

public function getActivityConfig()
{
   return $this->activityConfig;
}

```

#### 2，Replace Data Value with Object 对象取代数据值
> 一个数据项，需要与其他数据和行为一起使用才有意义。将数据项改成对象。随着设计深入，数据之间的关系逐渐显现出来，就需要将相关数据及其操作封装成对象。
```
ES中使用对象参数的场景较少。但如果当需要传递很多参数时，可以将多个组合成一个数组进行传递。
修改前：
public function push($name, array $body = array(), $timestamp = 0, $level = 'normal')
{
   ......
}

修改后：
public function push($cloudData)
{
   ......
}

```

#### 3，Replace Array with Object 以对象取代数组
> 数组中的元素各自代表不同的东西。以对象替换数组，对于数组中的每个元素，以一个字段来表示。
```
ES中很少用到对象参数.
```

#### 4，Replace Magic Number with Symbolic Constant字面常量取代魔法数
> 你有一个字面数值，带有特别含义。创建一个常量，根据其意义为它命名，并将上述的字面数值替换为这个常量。
```
修改前：
$this->getNoteDao()->search(array('courseId' => $courseId), array(), 0, 999999);

修改后：
$this->getNoteDao()->search(array('courseId' => $courseId), array(), 0, PHP_INT_MAX);
```

#### 5，Encapsulate Field 封装字段
> 你的类中存在一个public字段。将它声明为private，并提供相应的访问函数。
```
修改前：
public $productType = [
   'course'=> ['isPlugin' => false, 'imageUrlField' => ['cover' => ['large', 'middle', 'small']]],
   'classroom' => ['isPlugin' => false, 'imageUrlField' => ['smallPicture', 'middlePicture', 'largePicture']],
   'vip' => ['isPlugin' => true, 'pluginName' => 'Vip', 'imageUrlField' => []]
];

$this->productType;

修改后：

private $productType = [
   'course'=> ['isPlugin' => false, 'imageUrlField' => ['cover' => ['large', 'middle', 'small']]],
   'classroom' => ['isPlugin' => false, 'imageUrlField' => ['smallPicture', 'middlePicture', 'largePicture']],
   'vip' => ['isPlugin' => true, 'pluginName' => 'Vip', 'imageUrlField' => []]
];

$this->getProductType();

private function getProductType()
{

}

```

## 五，简化表达式

#### 1，Decompose Conditional 分解条件表达式
> 程序中，复杂的条件逻辑是最常导致复杂度上升的地点之一。可以将它分解为多个独立函数，根据每个小块代码的用途，为分解的新函数命名，从而更清楚的表达意图。
```
修改前：
public function generateEmail($registration, $maxLoop = 100)
{
   for ($i = 0; $i < $maxLoop; ++$i) {
      $registration['email'] = 'user_'.substr(base_convert(sha1(uniqid(mt_rand(), true)), 16, 36), 0, 9).'@edusoho.net';

      if ($this->isEmailAvaliable($registration['email'])) {
            break;
      }
   }

   return $registration['email'];
}

修改后：
public function generateEmail($registration, $maxLoop = 100)
{
   for ($i = 0; $i < $maxLoop; ++$i) {
      $registration['email'] = 'user_'.substr($this->getRandomChar(), 0, 9).'@edusoho.net';

      if ($this->isEmailAvaliable($registration['email'])) {
            break;
      }
   }

   return $registration['email'];
}

protected function getRandomChar()
{
   return base_convert(sha1(uniqid(mt_rand(), true)), 16, 36);
}
```

#### 2，Consolidate Conditional Expression 合并条件表达式
> 一系列条件测试，都得到相同结果。将这些测试合并为一个条件表达式，并将这个条件表达式提炼为一个独立函数。

#### 3，Consolodate Duplicate Conditional Fragments 合并重复的条件片段
> 在条件表达式的每个分支上有着相同的一段代码。将这段重复代码移到条件表达式之外。

#### 4，Remove Control Flag 移除控制标记
> 以break或return语句取代控制标记。

#### 5，Replace Nested Conditional with Guard Clauses 以卫语句取代嵌套条件表达式
```
修改前：
foreach ($fields as $field) {
   if ('studentNum' === $field) {
         $updateFields['studentNum'] = $this->countStudentsByCourseId($id);
   } elseif ('taskNum' === $field) {
         $updateFields['taskNum'] = $this->getTaskService()->countTasks(
            array('courseId' => $id, 'isOptional' => 0)
         );
   } elseif ('compulsoryTaskNum' === $field) {
         $updateFields['compulsoryTaskNum'] = $this->getTaskService()->countTasks(
            array('courseId' => $id, 'isOptional' => 0)
         );
   } elseif ('discussionNum' === $field) {
         $updateFields['discussionNum'] = $this->countThreadsByCourseIdAndType($id, 'discussion');
   } elseif ('questionNum' === $field) {
         $updateFields['questionNum'] = $this->countThreadsByCourseIdAndType($id, 'question');
   } elseif ('ratingNum' === $field) {
         $ratingFields = $this->getReviewService()->countRatingByCourseId($id);
         $updateFields = array_merge($updateFields, $ratingFields);
   } elseif ('noteNum' === $field) {
         $updateFields['noteNum'] = $this->getNoteService()->countCourseNoteByCourseId($id);
   } elseif ('materialNum' === $field) {
         $updateFields['materialNum'] = $this->getCourseMaterialService()->countMaterials(
            array('courseId' => $id, 'source' => 'coursematerial')
         );
   } elseif ('publishLessonNum' === $field) {
         $updateFields['publishLessonNum'] = $this->getCourseLessonService()->countLessons(
            array('courseId' => $id, 'status' => 'published')
         );
   } elseif ('lessonNum' === $field) {
         $updateFields['lessonNum'] = $this->getCourseLessonService()->countLessons(
            array('courseId' => $id)
         );
   }
}
修改后：
foreach ($fields as $field) {
   if ('studentNum' === $field) {
      $updateFields['studentNum'] = $this->countStudentsByCourseId($id);
   }

   if ('taskNum' === $field) {
      $updateFields['taskNum'] = $this->getTaskService()->countTasks(
         array('courseId' => $id, 'isOptional' => 0)
      );
   }

   if ('compulsoryTaskNum' === $field) {
      $updateFields['compulsoryTaskNum'] = $this->getTaskService()->countTasks(
         array('courseId' => $id, 'isOptional' => 0)
      );
   }

   if ('discussionNum' === $field) {
      $updateFields['discussionNum'] = $this->countThreadsByCourseIdAndType($id, 'discussion');
   }

   if ('questionNum' === $field) {
      $updateFields['questionNum'] = $this->countThreadsByCourseIdAndType($id, 'question');
   }

   ......
}

```

## 六，简化函数调用

#### 1，Rename Method 函数改名
> 给函数起一个好名字
```
修改前：
private function canBecomeClassroomMember($member)
{
   return empty($member) || !in_array('student', $member['role']);
}

修改后：
private function allowJoinClassroom($member)
{
   return empty($member) || !in_array('student', $member['role']);
}
```

#### 2，Add Parameter 添加参数
> 某个函数需要从调用端得到更多信息。为此函数添加一个对象参数，让该对象带进函数所需信息。

#### 3，Remove Parameter（移除参数）
> 函数不再需要某个参数时，将其移除。                                                                                                                                            
#### 4，Separate Query from Modifier 将查询函数和修改函数分离

#### 5，Parameterize Method 令函数携带参数
> 若干函数做了类似的工作，但在函数本体中却包含了不同的值。建立一个单一函数，以参数表达那些不同的值。

#### 6，Replace Parameter with Explicit Methods 以明确函数取代参数
> 某个函数完全取决于参数值而采取不同行为，为了获得一个清晰的接口，针对该参数的每一个可能值，建立一个独立函数。
```
修改前：
public function updateCourse($id, array('status' => 'close'));
public function updateCourse($id, array('status' => 'publish'));

修改后：
public function closeCourse($id);
public function publishCourse($id);
```

#### 7，Preserve whole object 保持对象完整
> 如果从对象中取出若干值，将它们作为某一次函数调用时的参数。改为传递整个对象。
> 除了可以使参数列更稳固外，还能简化参数列表，提高代码的可读性。
> 此外，使用完整对象，被调用函数可以利用完整对象中的函数来计算某些中间值。

#### 8，Introduce Parameter Object 引入参数对象
> 对象调用某个函数，并将所得结果作为参数，传递给另一个函数。而接受该参数的函数本身也能够调用前一个函数。让参数接受者去除该项参数，并直接调用前一个函数。

#### 9，Replace Error Code with Exception 以异常取代错误码

## 七，处理概况关系

#### 1，Pull Up Field 字段上移
> 两个子类拥有相同的字段。将该字段移至超类。

#### 2，Pull up Method 函数上移
> 有些函数，在各个子类中产生完全相同的结果。将该函数移至超类。

#### 3，Pull up Constructor Body 构造函数本体上移
> 各个子类中拥有一些极造函数，它们的本体几乎完全一致。在超类中新建一个构造函数，并在子类构造函数中调用它。

#### 4，Push down Method 函数下移
> 超类中的某个函数只与部分子类有关。将这个函数移到相关的那些子类去。

#### 5，Push down Fiedld 字段下移
> 超类中的某个字段只被部分子类用到，将这个字段移到需要它的那些子类去。

#### 6，Extract Subclass 提炼子类
> 类中的某些特性只被某些实例用到。新建一个子类，将这部分特性移到子类中。

#### 7，Extract Superclass 提炼超类
> 两个类有相似特性。为这2个类建立一个超类，将相同特性移至超类。