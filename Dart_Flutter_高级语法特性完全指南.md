# Dart/Flutter 高级语法特性完全指南

## 文档信息

- **版本**：v1.0（基于 Dart 3.3 / Flutter 3.19+）
- **最后更新**：2026-01-05
- **目标读者**：中级 Dart/Flutter 开发者

---

## 前置知识要求

在阅读本文档前，您应该已经掌握以下知识：

### 必备知识

- ✅ Dart 基础语法（变量、函数、控制流）
- ✅ 面向对象编程基础（类、对象、封装）
- ✅ 继承和接口的基本概念
- ✅ 泛型的基本使用（如 `List<String>`、`Map<K, V>`）
- ✅ 函数作为一等公民（函数参数、返回值、闭包）

### 推荐知识

- 📌 Flutter 基础开发经验
- 📌 了解类型系统的基本概念
- 📌 阅读过 Dart 官方文档的语言指南

### 不需要的知识

- ❌ 不需要深入理解编译器原理
- ❌ 不需要其他语言（Java/Kotlin/Swift）的经验
- ❌ 不需要函数式编程背景

---

## 文档使用说明

本文档采用**独立章节模式**，每章相对独立，您可以：

- 📖 按顺序阅读，逐步深入理解各个高级特性
- 🔍 跳转到感兴趣的章节作为参考手册使用
- 🔗 通过章节间的交叉引用了解特性之间的关联

### 推荐阅读顺序

- 如果您是首次学习这些特性，建议按章节顺序阅读（1→2→3→4→5→6→7→8→9）
- 如果您已有部分了解，可以直接跳转到需要的章节
- 第10章"综合实践案例"展示多特性协同，建议在阅读完前9章后查看

---

## 目录

1. [泛型约束与协变/逆变](#1-泛型约束与协变逆变)
2. [Mixin 多继承机制](#2-mixin-多继承机制)
3. [扩展方法（Extension Methods）](#3-扩展方法extension-methods)
4. [运算符重载（Operator Overloading）](#4-运算符重载operator-overloading)
5. [可调用类（Callable Classes）](#5-可调用类callable-classes)
6. [Records（记录类型）](#6-records记录类型)
7. [Pattern Matching 与解构](#7-pattern-matching-与解构)
8. [Sealed Classes & Enhanced Enums](#8-sealed-classes--enhanced-enums)
9. [Extension Types（扩展类型）](#9-extension-types扩展类型)
10. [综合实践案例](#10-综合实践案例)
11. [总结与最佳实践](#11-总结与最佳实践)

---

## 1. 泛型约束与协变/逆变

### 1.1 泛型基础回顾

泛型（Generics）是 Dart 类型系统的核心特性，允许我们编写类型安全且可复用的代码。在深入高级特性前，让我们快速回顾泛型的基本概念：

```dart
// 基础泛型使用
List<String> names = ['Alice', 'Bob'];
Map<int, User> userMap = {1: User('Alice')};
```

Dart 的泛型是**具化的**（reified），这意味着类型信息在运行时被保留，可以通过 `is` 检查：

```dart
void checkType<T>(List<T> list) {
  print(list is List<String>); // 运行时可以检查具体类型
}
```

### 1.2 泛型约束（Type Bounds）

#### 什么是泛型约束？

泛型约束使用 `extends` 关键字限制类型参数必须是某个类型或其子类型。这提供了更强的类型安全性，并允许我们在泛型代码中访问约束类型的成员。

#### 基本语法

```dart
class Container<T extends SomeBase> {
  // T 必须是 SomeBase 或其子类
  // 可以安全地调用 SomeBase 的方法
}
```

#### 为什么需要泛型约束？

1. **类型安全**：确保只有符合要求的类型可以使用
2. **成员访问**：可以调用约束类型的方法和属性
3. **语义明确**：代码意图更清晰

#### 实战示例：泛型 Repository 模式

以下示例展示如何使用泛型约束实现一个类型安全的数据仓库：

```dart
/// 所有实体的抽象基类
/// 提供统一的 id 属性，便于仓库进行查询
abstract class Entity {
  final int id;
  const Entity(this.id);
}

/// 用户实体
class User extends Entity {
  final String name;
  const User(int id, this.name) : super(id);

  @override
  String toString() => 'User(id: $id, name: $name)';
}

/// 商品实体
class Product extends Entity {
  final String title;
  final double price;
  const Product(int id, this.title, this.price) : super(id);

  @override
  String toString() => 'Product(id: $id, title: $title, price: $price)';
}

/// 泛型仓库类
/// T extends Entity 确保只能存储实体类型
/// 这样我们可以安全地访问 item.id
class Repository<T extends Entity> {
  final List<T> _items = [];

  /// 添加实体到仓库
  void add(T item) {
    _items.add(item);
    print('已添加: $item');
  }

  /// 根据 id 查找实体
  /// 因为 T extends Entity，所以可以访问 item.id
  T? findById(int id) {
    for (final item in _items) {
      if (item.id == id) {
        return item;
      }
    }
    return null;
  }

  /// 获取所有实体
  List<T> getAll() => List.unmodifiable(_items);

  /// 获取实体数量
  int get count => _items.length;
}

/// 使用示例
void main() {
  // 创建用户仓库
  final userRepo = Repository<User>();
  userRepo.add(const User(1, 'Alice'));
  userRepo.add(const User(2, 'Bob'));

  // 创建商品仓库
  final productRepo = Repository<Product>();
  productRepo.add(const Product(101, '机械键盘', 599.0));
  productRepo.add(const Product(102, '鼠标', 199.0));

  // 类型安全的查询
  final user = userRepo.findById(1);
  print('查询到用户: $user');

  final product = productRepo.findById(101);
  print('查询到商品: $product');

  // ❌ 编译错误：String 不是 Entity 的子类型
  // final stringRepo = Repository<String>(); // 无法编译
}
```

**关键要点**：
- ✅ `T extends Entity` 确保类型安全
- ✅ 可以在泛型类中访问 `Entity` 的成员（如 `id`）
- ✅ 编译器会阻止使用不符合约束的类型
- 📌 如果没有约束，`Repository<T>` 中无法访问 `item.id`

### 1.3 协变（Covariance）

#### 什么是协变？

协变描述的是类型参数的"方向"关系：如果 `Dog` 是 `Animal` 的子类型，那么在协变的情况下，`Container<Dog>` 也应该是 `Container<Animal>` 的子类型。

#### Dart 中的协变行为

在 Dart 中，**泛型集合类型是不变的**（invariant），这是一个重要的类型安全特性：

```dart
class Animal {}
class Dog extends Animal {}

void example() {
  List<Dog> dogs = [Dog()];
  // ❌ 编译错误：List<Dog> 不是 List<Animal> 的子类型
  // List<Animal> animals = dogs; // 无法编译
}
```

⚠️ **为什么集合不支持协变？**

如果允许 `List<Dog>` 赋值给 `List<Animal>`，会导致类型安全问题：

```dart
// 假设这是允许的（实际上不允许）
List<Dog> dogs = [Dog()];
List<Animal> animals = dogs; // 假设可以
animals.add(Cat()); // 添加一只猫
Dog dog = dogs[0]; // 💥 运行时错误！实际上是 Cat
```

#### covariant 关键字

Dart 提供了 `covariant` 关键字，允许在方法参数中使用协变，但这会将类型检查从编译时推迟到运行时：

```dart
class AnimalHandler {
  void handle(Animal animal) {
    print('处理动物');
  }
}

class DogHandler extends AnimalHandler {
  @override
  void handle(covariant Dog dog) {
    // 使用 covariant 缩小参数类型
    dog.bark(); // 可以调用 Dog 特有的方法
  }
}
```

#### 实战示例：covariant 的运行时风险

以下示例展示 `covariant` 如何将类型检查推迟到运行时，以及可能导致的问题：

```dart
/// 动物基类
class Animal {
  final String name;
  Animal(this.name);

  @override
  String toString() => '$runtimeType($name)';
}

/// 狗类
class Dog extends Animal {
  Dog(String name) : super(name);

  void bark() {
    print('$name 在叫：汪汪汪！');
  }
}

/// 猫类
class Cat extends Animal {
  Cat(String name) : super(name);

  void meow() {
    print('$name 在叫：喵喵喵！');
  }
}

/// 动物列表管理器
class AnimalList {
  final List<Animal> _animals = [];

  void addAnimal(Animal animal) {
    print('添加动物: $animal');
    _animals.add(animal);
  }

  List<Animal> get animals => List.unmodifiable(_animals);
}

/// 狗列表管理器
/// 使用 covariant 缩小参数类型
class DogList extends AnimalList {
  @override
  void addAnimal(covariant Dog dog) {
    // 因为参数类型是 Dog，可以调用 Dog 特有的方法
    dog.bark();
    super.addAnimal(dog);
  }
}

/// 演示 covariant 的运行时风险
void main() {
  // 通过父类型引用子类实例
  final AnimalList list = DogList();

  // ✅ 添加狗：编译通过，运行正常
  list.addAnimal(Dog('Lucky'));

  // ⚠️ 添加猫：编译通过，但运行时抛出类型错误！
  try {
    list.addAnimal(Cat('Kitty'));
  } catch (e) {
    print('💥 运行时错误: $e');
    print('原因：covariant 将类型检查推迟到运行时');
  }

  print('\n当前动物列表: ${list.animals}');
}

/// 输出：
/// Lucky 在叫：汪汪汪！
/// 添加动物: Dog(Lucky)
/// 💥 运行时错误: type 'Cat' is not a subtype of type 'Dog' in type cast
/// 原因：covariant 将类型检查推迟到运行时
/// 当前动物列表: [Dog(Lucky)]
```

**关键要点**：
- ⚠️ `covariant` 牺牲了编译时类型安全，换取了更灵活的类型关系
- ⚠️ 通过父类型引用时，编译器无法检测到类型不匹配
- ⚠️ 只有在运行时调用方法时才会抛出类型错误
- 📌 谨慎使用 `covariant`，只在确实需要时使用
- 📌 Dart 不支持 Kotlin/Java 的 `out` 关键字来声明协变

### 1.4 逆变（Contravariance）

#### 什么是逆变？

逆变与协变相反：如果 `Dog` 是 `Animal` 的子类型，那么在逆变的情况下，`Handler<Animal>` 应该是 `Handler<Dog>` 的子类型。

#### Dart 中的逆变：函数参数

在 Dart 中，**函数参数是逆变的**。这是一个重要但容易被忽视的特性：

```dart
typedef AnimalHandler = void Function(Animal);
typedef DogHandler = void Function(Dog);

void handleAnimal(Animal animal) {
  print('处理动物: ${animal.name}');
}

void example() {
  AnimalHandler animalHandler = handleAnimal;

  // ✅ 逆变：接受 Animal 的函数可以赋值给接受 Dog 的函数类型
  DogHandler dogHandler = animalHandler;

  dogHandler(Dog('Buddy')); // 正常工作
}
```

#### 为什么函数参数是逆变的？

这符合类型安全的直觉：
- 如果一个函数能处理所有 `Animal`，那它当然也能处理 `Dog`（Dog 是 Animal 的子类）
- 反过来不成立：能处理 `Dog` 的函数不一定能处理所有 `Animal`（比如 Cat）

#### 实战示例：函数参数逆变

以下示例展示函数参数的逆变特性在实际场景中的应用：

```dart
/// 动物基类
class Animal {
  final String name;
  Animal(this.name);

  @override
  String toString() => 'Animal($name)';
}

/// 狗类
class Dog extends Animal {
  Dog(String name) : super(name);

  void bark() => print('$name: 汪汪汪！');

  @override
  String toString() => 'Dog($name)';
}

/// 定义函数类型别名
typedef AnimalHandler = void Function(Animal animal);
typedef DogHandler = void Function(Dog dog);

/// 通用的动物处理函数（接受任何 Animal）
void feedAnimal(Animal animal) {
  print('正在喂养 ${animal.name}');
  print('提供食物和水');
}

/// 专门的狗处理函数（只接受 Dog）
void trainDog(Dog dog) {
  print('正在训练 ${dog.name}');
  dog.bark();
  print('训练完成！');
}

/// 接受 DogHandler 作为参数的函数
void processDog(Dog dog, DogHandler handler) {
  print('\n--- 开始处理狗 ---');
  handler(dog);
  print('--- 处理完成 ---');
}

/// 演示函数参数逆变
void main() {
  final dog = Dog('Buddy');

  // ✅ 逆变：AnimalHandler 可以赋值给 DogHandler
  // 因为能处理 Animal 的函数，一定能处理 Dog
  AnimalHandler genericHandler = feedAnimal;
  DogHandler specificHandler = genericHandler; // 逆变赋值

  // 使用逆变后的处理器
  processDog(dog, specificHandler);

  // ❌ 反向不成立：DogHandler 不能赋值给 AnimalHandler
  // 因为只能处理 Dog 的函数，不一定能处理所有 Animal
  DogHandler dogOnlyHandler = trainDog;
  // AnimalHandler generalHandler = dogOnlyHandler; // 编译错误

  // 直接传递也可以（类型推断）
  processDog(dog, feedAnimal); // ✅ 编译器自动处理逆变
}

/// 输出：
/// --- 开始处理狗 ---
/// 正在喂养 Buddy
/// 提供食物和水
/// --- 处理完成 ---
///
/// --- 开始处理狗 ---
/// 正在喂养 Buddy
/// 提供食物和水
/// --- 处理完成 ---
```

**关键要点**：
- ✅ 函数参数是逆变的：`void Function(Animal)` 可以赋值给 `void Function(Dog)`
- ✅ 这符合里氏替换原则：父类型的处理器可以替代子类型的处理器
- ❌ 反向不成立：`void Function(Dog)` 不能赋值给 `void Function(Animal)`
- 📌 函数返回值是协变的（与参数相反）
- 📌 Dart 不支持 Kotlin/Java 的 `in` 关键字来声明逆变

### 1.5 实际应用场景

#### 场景1：泛型工厂模式

使用泛型约束实现类型安全的工厂：

```dart
abstract class Serializable {
  Map<String, dynamic> toJson();
}

class SerializableFactory<T extends Serializable> {
  final T Function(Map<String, dynamic>) constructor;

  SerializableFactory(this.constructor);

  T fromJson(Map<String, dynamic> json) => constructor(json);

  List<T> fromJsonList(List<Map<String, dynamic>> jsonList) {
    return jsonList.map((json) => constructor(json)).toList();
  }
}
```

#### 场景2：类型安全的数据转换

使用泛型约束确保转换的类型安全：

```dart
class DataConverter<TInput extends Object, TOutput extends Object> {
  final TOutput Function(TInput) converter;

  DataConverter(this.converter);

  TOutput convert(TInput input) => converter(input);

  List<TOutput> convertList(List<TInput> inputs) {
    return inputs.map(converter).toList();
  }
}
```

### 1.7 Flutter 实战：类型安全的 ViewModel Provider

```dart
import 'package:flutter/material.dart';

class ViewModelProvider<T extends ChangeNotifier> extends InheritedNotifier<T> {
  const ViewModelProvider({
    super.key,
    required T notifier,
    required Widget child,
  }) : super(notifier: notifier, child: child);

  static T of<T extends ChangeNotifier>(BuildContext context) {
    final provider = context.dependOnInheritedWidgetOfExactType<ViewModelProvider<T>>();
    assert(provider != null, '未找到 ViewModelProvider<$T>');
    return provider!.notifier!;
  }
}

class CartCounter extends ChangeNotifier {
  int count = 0;
  void add() {
    count++;
    notifyListeners();
  }
}

class CartBadge extends StatelessWidget {
  const CartBadge({super.key});

  @override
  Widget build(BuildContext context) {
    final counter = ViewModelProvider.of<CartCounter>(context);
    return AnimatedBuilder(
      animation: counter,
      builder: (context, _) => Chip(label: Text('购物车: ${counter.count}')),
    );
  }
}

class CartPage extends StatelessWidget {
  const CartPage({super.key});

  @override
  Widget build(BuildContext context) {
    return ViewModelProvider<CartCounter>(
      notifier: CartCounter(),
      child: Builder(
        builder: (context) => Scaffold(
          appBar: AppBar(title: const Text('泛型 Provider')),
          floatingActionButton: FloatingActionButton(
            onPressed: () => ViewModelProvider.of<CartCounter>(context).add(),
            child: const Icon(Icons.add),
          ),
          body: const Center(child: CartBadge()),
        ),
      ),
    );
  }
}
```

**要点**：`ViewModelProvider` 使用 `T extends ChangeNotifier` 泛型约束，Flutter Widget 在编译期即可获得类型安全的 ViewModel。

### 1.8 常见陷阱与最佳实践

#### ⚠️ 陷阱1：误以为泛型集合支持协变

```dart
// ❌ 错误：List<Dog> 不是 List<Animal> 的子类型
List<Dog> dogs = [Dog()];
// List<Animal> animals = dogs; // 编译错误
```

**最佳实践**：如果需要协变行为，使用不可变集合或明确转换：

```dart
// ✅ 正确：显式创建新的 Animal 列表或不可变视图
final animals = List<Animal>.from(dogs);
// 或使用不可变视图（只读，更安全）
final animalsReadonly = List<Animal>.unmodifiable(dogs);

// ⚠️ 避免使用 cast()：它只提供运行时检查，写入时仍可能抛出 CastError
// final animals = dogs.cast<Animal>(); // 不推荐
```

#### ⚠️ 陷阱2：滥用 covariant 导致运行时错误

```dart
// ❌ 危险：过度使用 covariant
class Handler {
  void process(covariant dynamic data) {
    // 失去了类型安全
  }
}
```

**最佳实践**：避免使用 covariant，或采用其他设计模式：

```dart
// ✅ 方案1：避免使用 covariant，保持类型一致
class DogHandler extends AnimalHandler {
  @override
  void handle(Animal animal) {
    if (animal is Dog) {
      animal.bark();
    }
  }
}

// ✅ 方案2：使用泛型而不是 covariant
class TypedHandler<T extends Animal> {
  void handle(T animal) {
    // 类型安全的处理
  }
}

// ✅ 方案3：如果必须使用 covariant，确保调用方使用具体类型
class DogList {
  void addDog(Dog dog) {
    // 通过方法名和类型明确约束
  }
}
```

⚠️ **重要提示**：Dart 会在方法体执行前插入运行时类型转换，因此无法在方法内部捕获类型错误。类型检查失败会在调用边界抛出 `TypeError`。

#### ⚠️ 陷阱3：忽略函数参数的逆变特性

```dart
// ❌ 错误：试图将 DogHandler 赋值给 AnimalHandler
void Function(Dog) dogHandler = (dog) => dog.bark();
// void Function(Animal) animalHandler = dogHandler; // 编译错误
```

**最佳实践**：理解并利用逆变特性：

```dart
// ✅ 正确：利用逆变传递更通用的处理器
void Function(Animal) animalHandler = (animal) => print(animal.name);
void Function(Dog) dogHandler = animalHandler; // 逆变赋值
```

#### 📌 最佳实践总结

1. **优先使用泛型约束**：明确类型要求，提高代码可读性
2. **避免过度使用 covariant**：只在确实需要时使用，并添加运行时检查
3. **理解不变性**：记住 `List<Dog>` 不是 `List<Animal>` 的子类型
4. **利用逆变**：在回调函数中使用更通用的类型
5. **使用 typedef**：为复杂的函数类型创建别名，提高可读性

### 1.9 对比表：协变 vs 逆变 vs 不变

| 特性 | 协变 (Covariance) | 逆变 (Contravariance) | 不变 (Invariance) |
|------|-------------------|----------------------|-------------------|
| **定义** | 子类型关系保持方向：`Dog` → `Animal` 则 `Container<Dog>` → `Container<Animal>` | 子类型关系反转方向：`Dog` → `Animal` 则 `Handler<Animal>` → `Handler<Dog>` | 类型必须完全匹配，不存在子类型关系 |
| **Dart 中的应用** | 函数返回值类型、只读属性、`covariant` 参数 | 函数参数类型 | 泛型集合（`List<T>`、`Map<K,V>`） |
| **关键字** | `covariant`（方法参数） | 无专用关键字（自动应用于函数参数） | 默认行为 |
| **类型安全** | 运行时检查（使用 `covariant` 时） | 编译时检查 | 编译时检查 |
| **示例** | `void handle(covariant Dog dog)` | `void Function(Animal)` 可赋值给 `void Function(Dog)` | `List<Dog>` 不能赋值给 `List<Animal>` |
| **使用场景** | 方法重写时缩小参数类型 | 回调函数、事件处理器 | 可变集合、数据容器 |
| **风险** | 可能导致运行时类型错误 | 类型安全，无风险 | 限制灵活性，但保证安全 |

**快速记忆**：
- 📌 **协变**：子类型关系保持方向（子→父，如 `Dog` → `Animal`）
- 📌 **逆变**：子类型关系反转方向（父→子，如 `Handler<Animal>` → `Handler<Dog>`）
- 📌 **不变**：类型必须精确匹配

---

## 2. Mixin 多继承机制

### 2.1 Mixin 概念与设计目的

#### 什么是 Mixin？

Mixin 是 Dart 提供的一种代码复用机制，允许在不使用传统继承的情况下，将一个类的功能"混入"到另一个类中。与单继承不同，一个类可以使用多个 mixin。

#### 为什么需要 Mixin？

传统的单继承存在局限性：

```dart
// ❌ 问题：无法同时继承多个类
class Bird extends Animal {}
class FlyingBird extends Bird, Flyable {} // 编译错误
```

Mixin 解决了这个问题：

```dart
// ✅ 解决方案：使用 mixin
class Bird extends Animal with Flyable, Singable {}
```

**Mixin 的优势**：
1. **代码复用**：避免重复代码
2. **多重组合**：一个类可以使用多个 mixin
3. **横切关注点**：适合日志、缓存、验证等功能
4. **灵活性**：比继承更灵活，比接口更实用

### 2.2 Mixin 声明与使用

#### 基本语法

```dart
// 声明 mixin
mixin MixinName {
  // 方法和属性
}

// 使用 mixin
class MyClass with MixinName {}

// 组合多个 mixin
class MyClass with Mixin1, Mixin2, Mixin3 {}
```

#### mixin 关键字

Dart 2.1+ 引入了 `mixin` 关键字，明确声明一个类型是 mixin：

```dart
// 使用 mixin 关键字
mixin Logger {
  void log(String message) {
    print('[${DateTime.now()}] $message');
  }
}

// ❌ mixin 不能被实例化
// final logger = Logger(); // 编译错误

// ✅ 只能通过 with 使用
class Service with Logger {}
```

#### on 约束

使用 `on` 关键字限制 mixin 只能应用于特定类型：

```dart
mixin AuthMixin on User {
  // 只能应用于 User 或其子类
  void checkPermission() {
    print('检查用户 ${this.id} 的权限');
  }
}
```

#### 实战示例：基础 Mixin 组合

以下示例展示如何使用多个 mixin 组合功能：

```dart
/// 日志记录功能 mixin
mixin Logger {
  void log(String message) {
    print('[${DateTime.now().toIso8601String()}] $message');
  }
}

/// 数据校验功能 mixin
mixin Validator {
  bool validate(String value) {
    final isValid = value.isNotEmpty && value.length >= 3;
    print('[Validator] "$value" 校验${isValid ? '通过' : '失败'}');
    return isValid;
  }
}

/// 基础业务类
class ServiceBase {
  void run() {
    print('ServiceBase: 执行核心业务逻辑');
  }
}

/// 通过 with 链组合多个 mixin
/// 应用顺序：先 Logger，再 Validator（从左到右）
/// 方法解析顺序：UserService -> Validator -> Logger -> ServiceBase（从右到左）
class UserService extends ServiceBase with Logger, Validator {
  void saveUser(String name) {
    log('准备保存用户');

    if (validate(name)) {
      run();
      log('用户保存成功: $name');
    } else {
      log('用户保存失败：校验未通过');
    }
  }
}

/// 使用示例
void main() {
  final service = UserService();

  // ✅ 成功案例
  service.saveUser('Alice');

  print('\n---\n');

  // ❌ 失败案例
  service.saveUser('AB'); // 长度不足
}

/// 输出：
/// [2026-01-05T...] 准备保存用户
/// [Validator] "Alice" 校验通过
/// ServiceBase: 执行核心业务逻辑
/// [2026-01-05T...] 用户保存成功: Alice
///
/// ---
///
/// [2026-01-05T...] 准备保存用户
/// [Validator] "AB" 校验失败
/// [2026-01-05T...] 用户保存失败：校验未通过
```

**关键要点**：
- ✅ 使用 `with` 关键字组合多个 mixin
- ✅ Mixin 应用顺序是从左到右，但方法解析顺序是从右到左
- ✅ 最右边的 mixin 优先级最高（会先被查找）
- ✅ 可以同时访问多个 mixin 的方法
- 📌 Mixin 不能有构造函数

### 2.3 Mixin 的线性化顺序

#### 什么是线性化顺序？

当一个类使用多个 mixin 时，Dart 会将它们"线性化"成一个单一的继承链。这个顺序决定了方法查找的优先级。

#### 方法解析顺序（MRO）

Dart 的 mixin 应用是按顺序层层展开的：

```dart
class Base {}
mixin A {}
mixin B {}
mixin C {}

class MyClass extends Base with A, B, C {}
```

**展开过程**：
```dart
// 等价于以下展开：
class _Tmp1 = Base with A;
class _Tmp2 = _Tmp1 with B;
class MyClass = _Tmp2 with C;
```

**线性化顺序**：`MyClass -> C -> B -> A -> Base -> Object`

**关键规则**：
1. 子类优先于父类
2. Mixin 按照 `with` 声明的**从右到左**顺序解析
3. 最右边的 mixin 优先级最高（会先被查找）
4. 每个 mixin 依次"包裹"前一个结果

#### super 调用链

当多个 mixin 有同名方法时，`super` 会按照线性化顺序调用：

```dart
mixin A {
  void method() {
    print('A');
    super.method(); // 调用下一个
  }
}

mixin B {
  void method() {
    print('B');
    super.method();
  }
}

class Base {
  void method() {
    print('Base');
  }
}

class MyClass extends Base with A, B {
  @override
  void method() {
    print('MyClass');
    super.method(); // 调用 B.method()
  }
}

void main() {
  MyClass().method();
}

/// 输出：
/// MyClass
/// B
/// A
/// Base
```

**调用链**：`MyClass -> B -> A -> Base`

#### 实战示例：on 约束的使用

以下示例展示如何使用 `on` 约束限制 mixin 的应用范围：

```dart
/// 认证用户基类
/// 提供基础的用户身份信息
class AuthenticatedUser {
  final String userId;
  final String username;

  AuthenticatedUser(this.userId, this.username);

  @override
  String toString() => 'User($userId, $username)';
}

/// 会话管理 mixin
/// 使用 on 约束，只能应用于 AuthenticatedUser 或其子类
mixin SessionMixin on AuthenticatedUser {
  DateTime? _lastActivity;

  void refreshSession() {
    _lastActivity = DateTime.now();
    print('SessionMixin: 刷新用户 $userId 的会话');
    print('最后活动时间: ${_lastActivity!.toIso8601String()}');
  }

  bool isSessionExpired({Duration timeout = const Duration(minutes: 30)}) {
    if (_lastActivity == null) return true;
    return DateTime.now().difference(_lastActivity!) > timeout;
  }

  void checkSession() {
    if (isSessionExpired()) {
      print('⚠️ 会话已过期，用户: $username');
    } else {
      print('✅ 会话有效，用户: $username');
    }
  }
}

/// 管理员用户类
/// ✅ 正确：继承自 AuthenticatedUser，可以使用 SessionMixin
class AdminUser extends AuthenticatedUser with SessionMixin {
  final List<String> permissions;

  AdminUser(super.userId, super.username, this.permissions);

  void performAdminAction(String action) {
    checkSession();
    if (!isSessionExpired()) {
      print('执行管理员操作: $action');
      refreshSession();
    }
  }
}

/// 访客用户类
/// ❌ 错误示例（注释掉以避免编译错误）
/*
class GuestUser with SessionMixin {
  // 编译错误：GuestUser 未继承 AuthenticatedUser
  // 'SessionMixin' can't be mixed onto 'Object' because 'Object'
  // doesn't implement 'AuthenticatedUser'
}
*/

/// 使用示例
void main() {
  final admin = AdminUser('admin-001', 'Alice', ['read', 'write', 'delete']);

  // 刷新会话
  admin.refreshSession();

  print('\n--- 执行操作 ---');
  admin.performAdminAction('删除用户');

  print('\n--- 检查会话状态 ---');
  admin.checkSession();
}

/// 输出：
/// SessionMixin: 刷新用户 admin-001 的会话
/// 最后活动时间: 2026-01-05T...
///
/// --- 执行操作 ---
/// ✅ 会话有效，用户: Alice
/// 执行管理员操作: 删除用户
/// SessionMixin: 刷新用户 admin-001 的会话
/// 最后活动时间: 2026-01-05T...
///
/// --- 检查会话状态 ---
/// ✅ 会话有效，用户: Alice
```

**关键要点**：
- ✅ `on` 约束确保 mixin 只能应用于特定类型
- ✅ Mixin 可以访问约束类型的成员（如 `userId`、`username`）
- ✅ 编译器会阻止将 mixin 应用于不符合约束的类
- 📌 `on` 约束提供了类型安全，避免运行时错误

### 2.4 Mixin 与继承/接口的区别

#### 对比分析

| 特性 | Mixin | 继承 (extends) | 接口 (implements) |
|------|-------|----------------|-------------------|
| **代码复用** | ✅ 提供实现 | ✅ 提供实现 | ❌ 只定义契约 |
| **多重使用** | ✅ 可以使用多个 | ❌ 只能继承一个 | ✅ 可以实现多个 |
| **构造函数** | ❌ 不能有 | ✅ 可以有 | ❌ 不相关 |
| **访问私有成员** | ⚠️ 仅同一 library | ✅ 仅同一 library | ❌ 不能 |
| **is-a 关系** | ⚠️ 弱关系 | ✅ 强 is-a 关系 | ✅ 契约关系 |
| **方法冲突** | 按线性化顺序解决 | 子类覆盖父类 | 必须全部实现 |

#### 何时使用 Mixin？

**✅ 适合使用 Mixin 的场景**：
1. **横切关注点**：日志、缓存、验证等功能
2. **可选功能**：不是核心功能，但可以增强类的能力
3. **多个类共享**：多个不相关的类需要相同的功能
4. **避免深层继承**：不想创建复杂的继承层次

**❌ 不适合使用 Mixin 的场景**：
1. **核心功能**：类的本质特征应该通过继承表达
2. **需要构造函数**：Mixin 不能有构造函数
3. **强 is-a 关系**：如果是明确的"是一个"关系，用继承

#### 示例对比

```dart
// ❌ 不好：滥用 mixin
mixin Animal {
  void breathe() => print('呼吸');
}
class Dog with Animal {} // Dog "是一个" Animal，应该用继承

// ✅ 好：合理使用 mixin
class Dog extends Animal {}
mixin Trainable {
  void train() => print('训练');
}
class ServiceDog extends Dog with Trainable {} // 训练是可选功能
```

#### 实战示例：方法冲突解决与 super 调用

以下示例展示当多个 mixin 有同名方法时，如何通过 super 调用链解决冲突：

```dart
/// 审计日志 mixin
mixin AuditTrail {
  void execute() {
    print('AuditTrail: 记录审计日志');
  }
}

/// 缓存层 mixin
mixin CacheLayer {
  @override
  void execute() {
    print('CacheLayer: 检查并写入缓存');
    super.execute(); // 调用链中的下一个
  }
}

/// 网络层 mixin
mixin NetworkLayer {
  @override
  void execute() {
    print('NetworkLayer: 发起网络请求');
    super.execute(); // 调用链中的下一个
  }
}

/// 核心仓库类
class RepositoryCore {
  void execute() {
    print('RepositoryCore: 访问数据库');
  }
}

/// 数据仓库类
/// with 链顺序：RepositoryCore <- AuditTrail <- CacheLayer <- NetworkLayer
/// 线性化顺序：DataRepository -> NetworkLayer -> CacheLayer -> AuditTrail -> RepositoryCore
class DataRepository extends RepositoryCore
    with AuditTrail, CacheLayer, NetworkLayer {}

/// 使用示例
void main() {
  print('=== 方法解析顺序演示 ===\n');

  final repo = DataRepository();
  repo.execute();

  print('\n--- 调用链说明 ---');
  print('1. DataRepository 没有重写 execute，调用 NetworkLayer.execute()');
  print('2. NetworkLayer.execute() 打印后调用 super.execute()');
  print('3. super 指向 CacheLayer.execute()');
  print('4. CacheLayer.execute() 打印后调用 super.execute()');
  print('5. super 指向 AuditTrail.execute()');
  print('6. AuditTrail.execute() 打印后没有调用 super（终止）');
  print('7. 注意：RepositoryCore.execute() 没有被调用！');
}

/// 输出：
/// === 方法解析顺序演示 ===
///
/// NetworkLayer: 发起网络请求
/// CacheLayer: 检查并写入缓存
/// AuditTrail: 记录审计日志
///
/// --- 调用链说明 ---
/// 1. DataRepository 没有重写 execute，调用 NetworkLayer.execute()
/// 2. NetworkLayer.execute() 打印后调用 super.execute()
/// 3. super 指向 CacheLayer.execute()
/// 4. CacheLayer.execute() 打印后调用 super.execute()
/// 5. super 指向 AuditTrail.execute()
/// 6. AuditTrail.execute() 打印后没有调用 super（终止）
/// 7. 注意：RepositoryCore.execute() 没有被调用！
```

**关键要点**：
- ✅ 方法解析按照线性化顺序：最右边的 mixin 优先
- ✅ `super` 调用链中的下一个，而不是父类
- ⚠️ 如果 mixin 不调用 `super`，调用链会中断
- 📌 要调用基类方法，最后一个 mixin 必须调用 `super`

**修正版本**（确保调用基类）：

```dart
mixin AuditTrail {
  void execute() {
    print('AuditTrail: 记录审计日志');
    super.execute(); // ✅ 调用 super 确保链条完整
  }
}

// 现在 RepositoryCore.execute() 会被调用
```

### 2.5 Dart 3+ 的类修饰符

Dart 3.0 引入了新的类修饰符，影响 mixin 的使用方式。

#### 类修饰符概览

| 修饰符 | 说明 | 对 Mixin 的影响 |
|--------|------|-----------------|
| `base` | 类只能被继承，不能被实现 | 可以作为 mixin 的 `on` 约束 |
| `interface` | 类只能被实现，不能被继承 | 不能作为 mixin 的基类 |
| `final` | 类不能被继承或实现 | 不能作为 mixin 的基类 |
| `sealed` | 类只能在同一库中被继承 | 可以作为 mixin 的 `on` 约束 |
| `mixin class` | 既可以作为类也可以作为 mixin | 提供更灵活的复用方式 |

#### 示例

```dart
// base 类：可以被继承，可以作为 mixin 约束
base class User {
  final String id;
  User(this.id);
}

mixin AuthMixin on User {
  // ✅ 可以使用 base 类作为约束
}

// interface 类：只能被实现
interface class Config {
  void setup();
}

// ❌ 不能继承 interface 类
// class MyConfig extends Config {} // 编译错误

// mixin class：既是类也是 mixin
mixin class Logger {
  void log(String msg) => print(msg);
}

// 可以作为类使用
final logger = Logger();

// 也可以作为 mixin 使用
class Service with Logger {}
```

### 2.6 实际应用场景

#### 场景1：日志和监控

```dart
mixin LoggerMixin {
  void logInfo(String message) => print('[INFO] $message');
  void logError(String error) => print('[ERROR] $error');
}

mixin PerformanceMonitor {
  Future<T> measurePerformance<T>(String operation, Future<T> Function() action) async {
    final start = DateTime.now();
    final result = await action();
    final duration = DateTime.now().difference(start);
    print('[$operation] 耗时: ${duration.inMilliseconds}ms');
    return result;
  }
}

class ApiService with LoggerMixin, PerformanceMonitor {
  Future<void> fetchData() async {
    return measurePerformance('fetchData', () async {
      logInfo('开始获取数据');
      await Future.delayed(Duration(milliseconds: 100));
      logInfo('数据获取完成');
    });
  }
}
```

#### 场景2：状态管理

```dart
class User {
  final String name;
  final int level;
  const User(this.name, this.level);

  @override
  String toString() => 'User(name: $name, level: $level)';
}

mixin StateMixin<T> {
  T? _state;

  bool get hasState => _state != null;

  T get state {
    if (_state == null) {
      throw StateError('State 尚未初始化，请先调用 setState');
    }
    return _state as T;
  }

  void setState(T newState) {
    _state = newState;
    onStateChanged(newState);
  }

  void onStateChanged(T state) {}
}

class UserStore with StateMixin<User> {
  UserStore(User initialUser) {
    setState(initialUser);
  }

  @override
  void onStateChanged(User user) {
    print('用户状态更新: ${user.name}, 等级: ${user.level}');
  }
}

void main() {
  final store = UserStore(const User('Alice', 1));
  store.setState(const User('Alice', 2));
  print('当前用户: ${store.state}');
}
```

#### 场景3：Flutter StatefulWidget 中的 Mixin 协作

```dart
import 'package:flutter/material.dart';

mixin LoadingOverlay on State<StatefulWidget> {
  bool _loading = false;

  Future<T> executeWithOverlay<T>(Future<T> Function() future) async {
    setState(() => _loading = true);
    try {
      return await future();
    } finally {
      if (mounted) {
        setState(() => _loading = false);
      }
    }
  }

  Widget buildOverlay(Widget child) {
    if (_loading) {
      return Stack(
        children: [
          child,
          const ColoredBox(
            color: Colors.black38,
            child: Center(child: CircularProgressIndicator()),
          ),
        ],
      );
    }
    return child;
  }
}

class ProfilePage extends StatefulWidget {
  const ProfilePage({super.key});

  @override
  State<ProfilePage> createState() => _ProfilePageState();
}

class _ProfilePageState extends State<ProfilePage> with LoadingOverlay {
  String _text = '尚未加载';

  Future<void> _refresh() async {
    await executeWithOverlay(() async {
      await Future.delayed(const Duration(milliseconds: 600));
      setState(() => _text = '加载时间: ${DateTime.now()}');
    });
  }

  @override
  Widget build(BuildContext context) {
    return buildOverlay(
      Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Text(_text),
            const SizedBox(height: 12),
            ElevatedButton(onPressed: _refresh, child: const Text('刷新')),
          ],
        ),
      ),
    );
  }
}
```

**要点**：通过 mixin 将加载遮罩逻辑分离，`State` 类只需组合即可获得多重能力。

### 2.7 常见陷阱与最佳实践

#### ⚠️ 陷阱1：Mixin 顺序错误

```dart
// ❌ 错误：顺序导致方法被意外覆盖
class MyClass with MixinA, MixinB {
  // MixinB 的方法会覆盖 MixinA 的同名方法
}
```

**最佳实践**：仔细考虑 mixin 的顺序，将更通用的 mixin 放在前面。

#### ⚠️ 陷阱2：忘记调用 super

```dart
// ❌ 错误：中断了调用链
mixin MyMixin {
  void method() {
    print('MyMixin');
    // 忘记调用 super.method()
  }
}
```

**最佳实践**：如果 mixin 重写了方法，记得调用 `super` 以保持调用链完整。

#### ⚠️ 陷阱3：Mixin 携带状态导致耦合

```dart
// ❌ 不好：mixin 携带过多状态
mixin HeavyStateMixin {
  final List<String> _cache = [];
  final Map<String, dynamic> _config = {};
  // 过多的状态使 mixin 难以维护
}
```

**最佳实践**：Mixin 应该保持轻量，主要提供行为而非状态。

#### 📌 最佳实践总结

1. **单一职责**：每个 mixin 只负责一个功能
2. **明确命名**：使用描述性名称（如 `LoggerMixin`、`CacheMixin`）
3. **文档化约束**：如果使用 `on` 约束，在注释中说明原因
4. **避免深层嵌套**：不要创建 mixin 的 mixin
5. **考虑顺序**：将更通用的 mixin 放在前面

### 2.8 对比表：Mixin vs 继承 vs 接口 vs 扩展方法

| 特性 | Mixin | 继承 (extends) | 接口 (implements) | 扩展方法 (Extension) |
|------|-------|----------------|-------------------|---------------------|
| **代码复用** | ✅ 提供实现 | ✅ 提供实现 | ❌ 只定义契约 | ✅ 提供实现 |
| **多重使用** | ✅ 可以使用多个 | ❌ 只能继承一个 | ✅ 可以实现多个 | ✅ 可以使用多个 |
| **访问私有成员** | ⚠️ 仅同一 library | ✅ 仅同一 library | ❌ 不能 | ❌ 不能 |
| **构造函数** | ❌ 不能有 | ✅ 可以有 | ❌ 不相关 | N/A |
| **多态支持** | ✅ 支持 | ✅ 支持 | ✅ 支持 | ❌ 静态解析 |
| **状态管理** | ✅ 可以有字段 | ✅ 可以有字段 | ❌ 不能有实现 | ❌ 不能有状态 |
| **适用场景** | 横切关注点、可选功能 | is-a 关系、核心功能 | 契约定义、多态 | 语法糖、工具方法 |
| **类型关系** | ⚠️ 弱 is-a 关系 | ✅ 强 is-a 关系 | ✅ 契约关系 | ❌ 无类型关系 |

**快速选择指南**：
- 📌 **使用继承**：当存在明确的 is-a 关系时（如 Dog is an Animal）
- 📌 **使用 Mixin**：当需要在多个不相关的类中共享功能时（如 Logger、Cache）
- 📌 **使用接口**：当需要定义契约但不提供实现时（如 Comparable、Serializable）
- 📌 **使用扩展方法**：当需要为现有类添加工具方法时（详见第3章）

---

## 3. 扩展方法（Extension Methods）

### 3.1 扩展方法概念

#### 什么是扩展方法？

扩展方法（Extension Methods）允许你为现有的类添加新方法，而无需修改原始类或创建子类。这是一种"非侵入式"的代码增强方式。

#### 为什么需要扩展方法？

**传统方式的局限**：

```dart
// ❌ 问题1：无法修改第三方库的类
// 无法为 String 添加新方法

// ❌ 问题2：创建工具类不够优雅
class StringUtils {
  static String capitalize(String str) {
    // ...
  }
}
// 使用：StringUtils.capitalize(myString)
```

**扩展方法的解决方案**：

```dart
// ✅ 解决方案：使用扩展方法
extension StringExtensions on String {
  String capitalize() {
    // ...
  }
}
// 使用：myString.capitalize()
```

**扩展方法的优势**：
1. **语法优雅**：链式调用，符合面向对象风格
2. **类型安全**：编译时检查，支持泛型
3. **代码组织**：相关功能集中管理
4. **不侵入原类**：不修改原始类定义

### 3.2 扩展方法语法

#### 基本语法

```dart
// 命名扩展
extension ExtensionName on TargetType {
  // 方法定义
  ReturnType methodName(parameters) {
    // 可以访问 this（指向目标对象）
  }
}

// 匿名扩展（不推荐，难以管理冲突）
extension on TargetType {
  // 方法定义
}
```

#### 命名扩展 vs 匿名扩展

```dart
// ✅ 推荐：命名扩展
extension StringHelpers on String {
  String reverse() => split('').reversed.join();
}

// ⚠️ 不推荐：匿名扩展
extension on String {
  String reverse() => split('').reversed.join();
}
```

**为什么推荐命名扩展**：
- 可以显式调用解决冲突
- 便于文档和维护
- 支持 `show`/`hide` 导入控制

#### 泛型扩展

```dart
// 泛型扩展语法
extension ListExtensions<T> on List<T> {
  T? get firstOrNull => isEmpty ? null : first;
}

// 为可空列表添加去除 null 的方法
extension NullableListExtensions<T> on List<T?> {
  List<T> whereNotNull() {
    final result = <T>[];
    for (final e in this) {
      if (e != null) {
        result.add(e);
      }
    }
    return result;
  }
}
```

**关键要点**：
- 📌 `whereNotNull()` 定义在 `List<T?>` 上，返回 `List<T>`
- 📌 这样类型系统可以保证结果已经非空
- 📌 如果定义在 `List<T>` 上，当 `T` 是非空类型时，`e != null` 恒为 `true`

#### 实战示例：为 String 添加工具方法

以下示例展示如何为 String 类添加常用的工具方法：

```dart
/// 字符串工具扩展
extension StringToolkit on String {
  /// 首字母大写，其余部分保持原样
  String capitalizeFirst() {
    if (isEmpty) return this;
    return '${this[0].toUpperCase()}${substring(1)}';
  }

  /// 验证邮箱格式
  bool get isValidEmail {
    final pattern = RegExp(r'^[\w\.-]+@[\w\.-]+\.[a-zA-Z]{2,}$');
    return pattern.hasMatch(trim());
  }

  /// 将多余空白统一为单个空格并去除首尾空格
  String collapseWhitespace() {
    return trim().split(RegExp(r'\s+')).join(' ');
  }

  /// 反转字符串
  String reverse() {
    return split('').reversed.join();
  }
}

/// 使用示例
void main() {
  // 首字母大写
  final name = 'alice';
  print('首字母大写: ${name.capitalizeFirst()}'); // Alice

  // 邮箱验证
  final email1 = 'user@example.com';
  final email2 = 'invalid-email';
  print('${email1} 是否有效: ${email1.isValidEmail}'); // true
  print('${email2} 是否有效: ${email2.isValidEmail}'); // false

  // 规整空白
  final messy = '  Flutter   Extensions\tare  powerful ';
  print('规整后: "${messy.collapseWhitespace()}"');
  // 输出: "Flutter Extensions are powerful"

  // 反转字符串
  final text = 'Dart';
  print('反转: ${text.reverse()}'); // traD

  // ✅ 链式调用
  final result = '  hello world  '
      .collapseWhitespace()
      .capitalizeFirst()
      .reverse();
  print('链式调用结果: $result'); // dlrow olleH
}
```

**关键要点**：
- ✅ 扩展方法可以访问 `this`（目标对象）
- ✅ 支持 getter 和普通方法
- ✅ 可以链式调用
- 📌 扩展方法是静态解析的，不支持多态

### 3.3 扩展方法的可见性与作用域

#### 导入和作用域

扩展方法的可见性由导入语句控制：

```dart
// 导入扩展
import 'string_extensions.dart';

// 使用 show 只导入特定扩展
import 'string_extensions.dart' show StringToolkit;

// 使用 hide 排除特定扩展
import 'string_extensions.dart' hide StringToolkit;

// 使用前缀避免冲突
import 'string_extensions.dart' as str_ext;
```

#### 静态解析特性

扩展方法是**静态解析**的，这意味着：

```dart
extension AnimalExtension on Animal {
  void describe() => print('Animal');
}

extension DogExtension on Dog {
  void describe() => print('Dog');
}

void main() {
  Animal animal = Dog();
  animal.describe(); // 输出: Animal（不是 Dog！）

  // 静态类型决定调用哪个扩展
  Dog dog = Dog();
  dog.describe(); // 输出: Dog
}
```

⚠️ **重要**：扩展方法不支持多态，调用哪个扩展取决于变量的静态类型，而不是运行时类型。

#### 实战示例：泛型扩展

以下示例展示如何为 List<T> 添加类型安全的实用方法：

```dart
/// 列表组合器扩展
extension ListCombinators<T> on List<T> {
  /// 返回第一个匹配元素，找不到返回 null
  T? firstWhereOrNull(bool Function(T) predicate) {
    for (final element in this) {
      if (predicate(element)) {
        return element;
      }
    }
    return null;
  }

  /// 映射后丢弃 null，常用于空值过滤
  List<R> mapNotNull<R>(R? Function(T) transform) {
    final result = <R>[];
    for (final element in this) {
      final mapped = transform(element);
      if (mapped != null) {
        result.add(mapped);
      }
    }
    return result;
  }

  /// 保留满足条件的元素并返回新列表
  List<T> filter(bool Function(T) predicate) {
    return where(predicate).toList();
  }

  /// 安全获取指定索引的元素
  T? getOrNull(int index) {
    if (index < 0 || index >= length) return null;
    return this[index];
  }
}

/// 使用示例
void main() {
  // 查找第一个偶数
  final numbers = [1, 2, 3, 4, 5];
  final firstEven = numbers.firstWhereOrNull((e) => e.isEven);
  print('第一个偶数: $firstEven'); // 2

  // 过滤奇数
  final odds = numbers.filter((e) => e.isOdd);
  print('奇数列表: $odds'); // [1, 3, 5]

  // mapNotNull：解析字符串为整数
  final rawValues = ['1', 'two', '3', 'four', '5'];
  final parsed = rawValues.mapNotNull<int>((value) => int.tryParse(value));
  print('解析结果: $parsed'); // [1, 3, 5]

  // 安全获取元素
  print('索引 2: ${numbers.getOrNull(2)}'); // 3
  print('索引 10: ${numbers.getOrNull(10)}'); // null

  // ✅ 类型安全：编译时检查
  final strings = ['a', 'b', 'c'];
  final firstLong = strings.firstWhereOrNull((s) => s.length > 1);
  // firstLong 的类型是 String?
}
```

**关键要点**：
- ✅ 泛型扩展提供类型安全
- ✅ 可以定义多个类型参数（如 `mapNotNull<R>`）
- ✅ 编译器会进行类型推断
- 📌 泛型约束也适用于扩展（如 `on List<T extends Comparable>`）

### 3.4 扩展方法的限制

#### 限制1：无法访问私有成员（跨库）

扩展方法无法访问目标类的私有成员（以 `_` 开头），**除非扩展与目标类在同一个 library 中**：

```dart
// ========== 文件: user.dart (library A) ==========
class User {
  final String name;
  final String _password; // 私有字段

  User(this.name, this._password);
}

// ========== 文件: user_extension.dart (library B) ==========
import 'user.dart';

extension UserExtension on User {
  void showInfo() {
    print('Name: $name'); // ✅ 可以访问公共成员
    // print('Password: $_password'); // ❌ 编译错误：无法访问私有成员
  }
}
```

**关键要点**：
- ⚠️ 私有成员的可见性是基于 **library**（库），不是基于类
- ✅ 如果扩展与目标类在同一个 library，可以访问私有成员
- ❌ 如果扩展与目标类在不同 library，无法访问私有成员

#### 限制2：静态解析，不支持多态

如前所述，扩展方法是静态解析的：

```dart
extension AnimalExt on Animal {
  void greet() => print('Animal');
}

extension DogExt on Dog {
  void greet() => print('Dog');
}

Animal animal = Dog();
animal.greet(); // 输出: Animal（不是 Dog）
```

#### 限制3：无法定义字段

扩展方法不能添加实例字段，只能添加方法和 getter：

```dart
extension BadExtension on String {
  // ❌ 编译错误：扩展不能有实例字段
  // String _cache;

  // ✅ 可以有 getter
  int get wordCount => split(' ').length;
}
```

#### 限制4：命名冲突

如果多个扩展定义了相同的方法名，会导致编译错误：

```dart
extension Ext1 on String {
  void process() => print('Ext1');
}

extension Ext2 on String {
  void process() => print('Ext2');
}

void main() {
  // ❌ 编译错误：二义性
  // 'hello'.process();

  // ✅ 解决方案：显式调用
  Ext1('hello').process();
  Ext2('hello').process();
}
```

#### 实战示例：命名空间管理与冲突解决

以下示例展示如何处理扩展方法的命名冲突：

```dart
// ========== 文件: classic_formatter.dart ==========
/// 经典格式化扩展
extension ClassicJsonFormatter on String {
  String prettyPrint() {
    // 简化的 JSON 美化（演示用）
    return replaceAll('{', '{\n  ')
        .replaceAll('}', '\n}')
        .replaceAll(',', ',\n  ');
  }
}

// ========== 文件: compact_formatter.dart ==========
/// 紧凑格式化扩展
extension CompactJsonFormatter on String {
  String prettyPrint() {
    // 移除所有空白
    return replaceAll(RegExp(r'\s+'), '');
  }
}

// ========== 文件: main.dart ==========
// 方案1：使用 hide 排除冲突的扩展
import 'classic_formatter.dart';
import 'compact_formatter.dart' hide CompactJsonFormatter;

// 方案2：使用前缀避免冲突
import 'compact_formatter.dart' as compact;

void main() {
  const json = '{ "name": "Alice", "age": 25 }';

  // ✅ 方案1：默认使用 ClassicJsonFormatter
  print('经典格式:');
  print(json.prettyPrint());

  // ✅ 方案2：显式调用 CompactJsonFormatter
  print('\n紧凑格式:');
  print(compact.CompactJsonFormatter(json).prettyPrint());

  // ✅ 方案3：显式调用特定扩展
  print('\n显式调用经典格式:');
  print(ClassicJsonFormatter(json).prettyPrint());
}

/// 输出：
/// 经典格式:
/// {
///   "name": "Alice",
///   "age": 25
/// }
///
/// 紧凑格式:
/// {"name":"Alice","age":25}
///
/// 显式调用经典格式:
/// {
///   "name": "Alice",
///   "age": 25
/// }
```

**关键要点**：
- ✅ 使用 `hide` 排除冲突的扩展
- ✅ 使用 `as` 前缀避免冲突
- ✅ 显式调用：`ExtensionName(object).method()`
- 📌 命名扩展是解决冲突的关键

### 3.5 实际应用场景

#### 场景1：为第三方库添加功能

```dart
// 为 DateTime 添加便捷方法
extension DateTimeExtensions on DateTime {
  bool get isToday {
    final now = DateTime.now();
    return year == now.year && month == now.month && day == now.day;
  }

  bool get isYesterday {
    final yesterday = DateTime.now().subtract(Duration(days: 1));
    return year == yesterday.year &&
           month == yesterday.month &&
           day == yesterday.day;
  }

  String toFriendlyString() {
    if (isToday) return '今天';
    if (isYesterday) return '昨天';
    return '$year-$month-$day';
  }
}
```

#### 场景2：语法糖封装

```dart
// 为 Future 添加便捷方法
extension FutureExtensions<T> on Future<T> {
  Future<T> withTimeout(Duration duration) {
    return timeout(duration, onTimeout: () {
      throw TimeoutException('操作超时');
    });
  }

  Future<R> thenMap<R>(R Function(T) mapper) {
    return then((value) => mapper(value));
  }
}
```

#### 场景3：Flutter `BuildContext` 快捷扩展

```dart
import 'package:flutter/material.dart';

extension ContextX on BuildContext {
  ThemeData get theme => Theme.of(this);
  TextTheme get textTheme => theme.textTheme;
  Size get screenSize => MediaQuery.of(this).size;
  void showSnack(String message) => ScaffoldMessenger.of(this).showSnackBar(
        SnackBar(content: Text(message)),
      );
}

class ThemeAwareButton extends StatelessWidget {
  const ThemeAwareButton({super.key});

  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      style: ElevatedButton.styleFrom(
        backgroundColor: context.theme.colorScheme.primary,
        minimumSize: Size(context.screenSize.width * 0.6, 48),
      ),
      onPressed: () => context.showSnack('点击时间: ${DateTime.now()}'),
      child: Text(
        '扩展方法示例',
        style: context.textTheme.titleMedium?.copyWith(color: Colors.white),
      ),
    );
  }
}
```

**要点**：通过扩展方法封装常用的 `Theme`、`MediaQuery`、`ScaffoldMessenger` 调用，使 Flutter Widget 更简洁。

### 3.6 常见陷阱与最佳实践

#### ⚠️ 陷阱1：误以为扩展方法支持多态

```dart
// ❌ 错误：期望多态行为
Animal animal = Dog();
animal.describe(); // 调用 AnimalExtension，不是 DogExtension
```

**最佳实践**：理解扩展方法是静态解析的，不要依赖多态。

#### ⚠️ 陷阱2：过度使用扩展方法

```dart
// ❌ 不好：为每个类都添加扩展
extension UserExtension on User {
  void save() {} // 核心功能应该在类内部
}
```

**最佳实践**：只为工具方法和语法糖使用扩展，核心功能应该在类内部实现。

#### ⚠️ 陷阱3：命名冲突管理不当

```dart
// ❌ 危险：导入多个有冲突的扩展
import 'ext1.dart';
import 'ext2.dart'; // 可能有冲突
```

**最佳实践**：使用 `show`/`hide` 明确管理导入，或使用前缀。

#### 📌 最佳实践总结

1. **命名清晰**：使用描述性的扩展名称（如 `StringHelpers`）
2. **单一职责**：每个扩展专注于一类功能
3. **避免副作用**：扩展方法应该是纯函数或只读操作
4. **文档化**：为扩展方法添加清晰的注释
5. **谨慎使用**：不要滥用扩展，保持代码简洁

### 3.7 对比表：扩展方法 vs Mixin vs 继承

| 特性 | 扩展方法 (Extension) | Mixin | 继承 (extends) |
|------|---------------------|-------|----------------|
| **代码复用** | ✅ 提供实现 | ✅ 提供实现 | ✅ 提供实现 |
| **多重使用** | ✅ 可以使用多个 | ✅ 可以使用多个 | ❌ 只能继承一个 |
| **访问私有成员** | ❌ 不能 | ⚠️ 仅同一 library | ✅ 仅同一 library |
| **添加字段** | ❌ 不能 | ✅ 可以 | ✅ 可以 |
| **多态支持** | ❌ 静态解析 | ✅ 支持 | ✅ 支持 |
| **修改原类** | ❌ 不修改 | ❌ 不修改 | ⚠️ 创建子类 |
| **适用场景** | 工具方法、语法糖 | 横切关注点、可选功能 | is-a 关系、核心功能 |
| **类型关系** | ❌ 无类型关系 | ⚠️ 弱 is-a 关系 | ✅ 强 is-a 关系 |

**快速选择指南**：
- 📌 **使用扩展方法**：为现有类添加工具方法，不需要修改原类
- 📌 **使用 Mixin**：在多个类中共享功能，需要访问实例状态
- 📌 **使用继承**：存在明确的 is-a 关系，需要多态支持

---

## 4. 运算符重载（Operator Overloading）

### 4.1 运算符重载概念

#### 什么是运算符重载？

运算符重载允许你为自定义类定义运算符（如 `+`、`-`、`*`、`==` 等）的行为。这使得自定义类型可以像内置类型一样使用运算符。

#### 为什么需要运算符重载？

**传统方式的局限**：

```dart
// ❌ 问题：使用方法调用不够直观
class Vector {
  double x, y;
  Vector(this.x, this.y);

  Vector add(Vector other) {
    return Vector(x + other.x, y + other.y);
  }
}

// 使用：v1.add(v2).add(v3) // 不够直观
```

**运算符重载的解决方案**：

```dart
// ✅ 解决方案：使用运算符重载
class Vector {
  double x, y;
  Vector(this.x, this.y);

  Vector operator +(Vector other) {
    return Vector(x + other.x, y + other.y);
  }
}

// 使用：v1 + v2 + v3 // 直观自然
```

**运算符重载的优势**：
1. **语法简洁**：符合数学和自然语言习惯
2. **代码可读**：表达意图更清晰
3. **类型安全**：编译时检查
4. **链式操作**：支持连续运算

### 4.2 可重载的运算符列表

#### Dart 支持重载的运算符

| 类别 | 运算符 |
|------|--------|
| **算术运算符** | `+`, `-`, `*`, `/`, `~/`, `%` |
| **关系运算符** | `<`, `>`, `<=`, `>=` |
| **相等运算符** | `==` |
| **位运算符** | `&`, `|`, `^`, `~`, `<<`, `>>`, `>>>` |
| **索引运算符** | `[]`, `[]=` |
| **一元运算符** | `-`, `~` |

#### 不能重载的运算符

以下运算符**不能**被重载：
- `?.` (空安全调用)
- `??` (空值合并)
- `is` (类型检查)
- `as` (类型转换)
- `&&`, `||` (逻辑运算符)
- `!` (逻辑非)

### 4.3 运算符重载语法

#### 基本语法

```dart
class ClassName {
  // 二元运算符
  ReturnType operator +(Type other) {
    // 实现
  }

  // 一元运算符
  ReturnType operator -() {
    // 实现
  }

  // 索引运算符
  ReturnType operator [](int index) {
    // 实现
  }

  void operator []=(int index, Type value) {
    // 实现
  }
}
```

#### 关键要点

- ✅ 使用 `operator` 关键字
- ✅ 运算符后面跟参数（二元运算符）或不跟参数（一元运算符）
- ✅ 可以有返回值
- 📌 运算符重载是实例方法，可以访问 `this`

#### 实战示例：2D向量的算术运算

以下示例展示如何为2D向量类重载算术运算符：

```dart
import 'dart:math';

/// 2D向量类
class Vector2D {
  final double x;
  final double y;

  const Vector2D(this.x, this.y);

  /// 向量加法：对应分量求和
  Vector2D operator +(Vector2D other) {
    return Vector2D(x + other.x, y + other.y);
  }

  /// 向量减法：对应分量做差
  Vector2D operator -(Vector2D other) {
    return Vector2D(x - other.x, y - other.y);
  }

  /// 标量乘法：与数字相乘
  Vector2D operator *(double scalar) {
    return Vector2D(x * scalar, y * scalar);
  }

  /// 一元负号：取反
  Vector2D operator -() {
    return Vector2D(-x, -y);
  }

  /// 向量长度
  double get length => sqrt(x * x + y * y);

  @override
  String toString() => 'Vector2D($x, $y)';
}

/// 使用示例
void main() {
  const v1 = Vector2D(3, 4);
  const v2 = Vector2D(1, 2);

  // 向量加法
  final sum = v1 + v2;
  print('v1 + v2 = $sum'); // Vector2D(4.0, 6.0)

  // 向量减法
  final diff = v1 - v2;
  print('v1 - v2 = $diff'); // Vector2D(2.0, 2.0)

  // 标量乘法
  final scaled = v1 * 2.5;
  print('v1 * 2.5 = $scaled'); // Vector2D(7.5, 10.0)

  // 一元负号
  final negated = -v1;
  print('-v1 = $negated'); // Vector2D(-3.0, -4.0)

  // ✅ 链式操作
  final result = (v1 + v2) * 2.0 - v1;
  print('(v1 + v2) * 2 - v1 = $result'); // Vector2D(5.0, 8.0)
}
```

**关键要点**：
- ✅ 运算符重载使代码更直观
- ✅ 支持链式操作
- ✅ 保持运算符的语义一致性（`+` 用于加法，`-` 用于减法）
- 📌 返回新对象，不修改原对象（不可变性）

### 4.4 特殊运算符

#### == 运算符与 hashCode

`==` 运算符是最常用的运算符之一，但它有特殊要求：**必须与 `hashCode` 配对实现**。

**为什么必须配对？**

Dart 的集合类（`Set`、`Map`）使用 `hashCode` 来优化查找性能。如果只重载 `==` 而不重载 `hashCode`，会导致集合行为异常。

**相等性的三个原则**：
1. **自反性**：`x == x` 必须为 `true`
2. **对称性**：如果 `x == y`，则 `y == x`
3. **传递性**：如果 `x == y` 且 `y == z`，则 `x == z`

#### 实战示例：== 与 hashCode 的正确实现

以下示例对比正确和错误的实现：

```dart
import 'dart:math';

/// ❌ 错误示例：仅重写 ==，未同步重写 hashCode
class BrokenPoint {
  final int x;
  final int y;

  const BrokenPoint(this.x, this.y);

  @override
  bool operator ==(Object other) =>
      other is BrokenPoint && other.x == x && other.y == y;

  // ⚠️ 未重写 hashCode，使用默认实现（基于对象标识）
  // 这会导致相等的对象有不同的 hashCode
}

/// ✅ 正确示例：== 与 hashCode 配对实现
class Point {
  final int x;
  final int y;

  const Point(this.x, this.y);

  @override
  bool operator ==(Object other) =>
      other is Point && other.x == x && other.y == y;

  @override
  int get hashCode => Object.hash(x, y);

  @override
  String toString() => 'Point($x, $y)';
}

/// 使用示例
void main() {
  print('=== 错误示例：BrokenPoint ===');
  final broken1 = const BrokenPoint(1, 2);
  final broken2 = const BrokenPoint(1, 2);

  print('broken1 == broken2: ${broken1 == broken2}'); // true

  final brokenSet = <BrokenPoint>{};
  brokenSet.add(broken1);
  brokenSet.add(broken2);
  print('BrokenPoint Set 大小: ${brokenSet.length}'); // 2（错误！应该是1）
  print('原因：hashCode 不同，Set 认为是不同对象\n');

  print('=== 正确示例：Point ===');
  final point1 = const Point(1, 2);
  final point2 = const Point(1, 2);

  print('point1 == point2: ${point1 == point2}'); // true
  print('hashCode 相同: ${point1.hashCode == point2.hashCode}'); // true

  final pointSet = <Point>{};
  pointSet.add(point1);
  pointSet.add(point2);
  print('Point Set 大小: ${pointSet.length}'); // 1（正确！）

  // Map 中的使用
  final pointMap = <Point, String>{};
  pointMap[const Point(0, 0)] = '原点';
  pointMap[const Point(0, 0)] = '覆盖后的值';
  print('Point Map: $pointMap'); // {Point(0, 0): 覆盖后的值}
}
```

**关键要点**：
- ⚠️ **必须配对**：重载 `==` 时必须同时重载 `hashCode`
- ✅ 使用 `Object.hash()` 计算 hashCode（Dart 2.14+）
- ✅ 相等的对象必须有相同的 hashCode
- ✅ `==` 的参数类型必须是 `Object`
- 📌 `hashCode` 应该基于用于相等性比较的字段计算

#### 索引运算符 [] 和 []=

索引运算符允许对象像数组一样通过下标访问元素。

```dart
/// 带边界检查的安全列表（仅支持非空类型）
class SafeList<T extends Object> {
  final List<T?> _items;

  /// 创建固定长度的容器
  SafeList(int length)
      : assert(length >= 0, 'length 必须为非负数'),
        _items = List<T?>.filled(length, null, growable: false);

  /// 重载 [] 读取元素
  T operator [](int index) {
    _checkBounds(index);
    final value = _items[index];
    if (value == null) {
      throw StateError('索引 $index 尚未初始化');
    }
    return value;
  }

  /// 重载 []= 写入元素
  void operator []=(int index, T value) {
    _checkBounds(index);
    _items[index] = value;
  }

  int get length => _items.length;

  void _checkBounds(int index) {
    if (index < 0 || index >= _items.length) {
      throw RangeError.index(
        index,
        _items,
        'index',
        '索引越界',
        _items.length,
      );
    }
  }

  @override
  String toString() => _items.toString();
}

void main() {
  final scores = SafeList<int>(3);

  // 写入数据
  scores[0] = 95;
  scores[1] = 88;

  print('第一门成绩: ${scores[0]}'); // 95
  print('SafeList 状态: $scores'); // [95, 88, null]

  // 错误处理示例
  try {
    print(scores[2]); // 未初始化，抛出 StateError
  } catch (e) {
    print('读取错误: $e');
  }

  try {
    scores[5] = 100; // 越界，抛出 RangeError
  } catch (e) {
    print('写入错误: $e');
  }
}
```

**关键要点**：
- ✅ `operator []` 用于读取，`operator []=` 用于写入
- ✅ 索引运算符可以有任意参数类型（不限于 int）
- ⚠️ 应该提供合理的边界检查和错误处理
- 📌 常用于自定义集合类型、矩阵、映射等数据结构

---

### 4.5 实际应用场景

#### 场景1：自定义数学类型

运算符重载最经典的应用是数学类型（向量、矩阵、复数等）：

```dart
class Complex {
  final double real;
  final double imaginary;

  const Complex(this.real, this.imaginary);

  Complex operator +(Complex other) =>
      Complex(real + other.real, imaginary + other.imaginary);

  Complex operator *(Complex other) =>
      Complex(
        real * other.real - imaginary * other.imaginary,
        real * other.imaginary + imaginary * other.real,
      );

  @override
  String toString() => '$real + ${imaginary}i';
}

void main() {
  final a = Complex(3, 2);
  final b = Complex(1, 4);
  print('a + b = ${a + b}'); // 4.0 + 6.0i
  print('a * b = ${a * b}'); // -5.0 + 14.0i
}
```

---

#### 场景2：领域特定语言（DSL）

运算符重载可以创建更自然的 DSL 语法：

```dart
class TimeSpan {
  final int milliseconds;

  const TimeSpan.milliseconds(this.milliseconds);
  const TimeSpan.seconds(int seconds) : milliseconds = seconds * 1000;
  const TimeSpan.minutes(int minutes) : milliseconds = minutes * 60 * 1000;

  TimeSpan operator +(TimeSpan other) =>
      TimeSpan.milliseconds(milliseconds + other.milliseconds);

  TimeSpan operator *(int factor) =>
      TimeSpan.milliseconds(milliseconds * factor);

  bool operator >(TimeSpan other) => milliseconds > other.milliseconds;
  bool operator <(TimeSpan other) => milliseconds < other.milliseconds;
  bool operator >=(TimeSpan other) => milliseconds >= other.milliseconds;
  bool operator <=(TimeSpan other) => milliseconds <= other.milliseconds;

  @override
  String toString() => '${milliseconds}ms';
}

void main() {
  final timeout = TimeSpan.seconds(5) + TimeSpan.milliseconds(500);
  final extended = timeout * 2;

  print('超时时间: $timeout'); // 5500ms
  print('延长后: $extended'); // 11000ms
  print('是否超过10秒: ${extended > TimeSpan.seconds(10)}'); // true
}
```

#### 场景3：不适合使用运算符重载的情况

⚠️ **反例**：不要为了"炫技"而滥用运算符重载

```dart
// ❌ 错误示例：语义不清晰
class User {
  final String name;
  User(this.name);

  // + 运算符用于"添加好友"？语义模糊！
  User operator +(User other) {
    print('$name 添加了好友 ${other.name}');
    return this;
  }
}

// ✅ 正确做法：使用明确的方法名
class BetterUser {
  final String name;
  BetterUser(this.name);

  void addFriend(BetterUser other) {
    print('$name 添加了好友 ${other.name}');
  }
}
```

---

### 4.6 常见陷阱与最佳实践

#### 陷阱1：忘记配对 == 和 hashCode

```dart
// ❌ 错误：仅重载 ==
class BadPoint {
  final int x, y;
  BadPoint(this.x, this.y);

  @override
  bool operator ==(Object other) =>
      other is BadPoint && other.x == x && other.y == y;
  // 缺少 hashCode！
}

void main() {
  final set = {BadPoint(1, 2), BadPoint(1, 2)};
  print(set.length); // 2（预期是1）- Set 无法正确去重
}
```

**最佳实践**：
- ✅ 始终同时重载 `==` 和 `hashCode`
- ✅ 使用 `Object.hash()` 计算 hashCode
- ✅ IDE 通常会提示缺少 hashCode

---

#### 陷阱2：运算符语义不一致

```dart
// ❌ 错误：+ 运算符修改了原对象
class BadCounter {
  int value = 0;

  BadCounter operator +(int increment) {
    value += increment; // 修改了自身！
    return this;
  }
}

void main() {
  final a = BadCounter();
  final b = a + 5;
  print('a.value: ${a.value}'); // 5（意外！）
  print('a == b: ${identical(a, b)}'); // true（同一对象）
}

// ✅ 正确：返回新对象，保持不可变性
class GoodCounter {
  final int value;
  const GoodCounter(this.value);

  GoodCounter operator +(int increment) =>
      GoodCounter(value + increment);
}

void main2() {
  final a = GoodCounter(0);
  final b = a + 5;
  print('a.value: ${a.value}'); // 0（符合预期）
  print('b.value: ${b.value}'); // 5
}
```

**最佳实践**：
- ✅ 运算符应该返回新对象，而不是修改原对象
- ✅ 保持不可变性（immutability）
- ✅ 遵循数学直觉（`a + b` 不应改变 `a` 或 `b`）
- ⚠️ 如果必须修改状态，使用明确的方法名（如 `increment()`）

---

### 4.7 运算符重载适用场景对比

| 场景类型 | 是否推荐 | 典型示例 | 原因 |
|---------|---------|---------|------|
| 数学类型 | ✅ 强烈推荐 | Vector、Matrix、Complex | 符合数学直觉，语义清晰 |
| 值对象 | ✅ 推荐 | Money、Duration、Range | 运算符语义明确，提高可读性 |
| 集合类型 | ✅ 推荐 | CustomList、SafeArray | 索引访问符合习惯 |
| 比较操作 | ✅ 推荐 | 任何需要排序的类 | `==`、`<`、`>` 等语义明确 |
| 业务逻辑 | ⚠️ 谨慎使用 | User、Order | 语义可能不清晰，优先使用方法 |
| 字符串拼接 | ❌ 不推荐 | `user + "suffix"` | 语义模糊，使用方法更清晰 |
| 副作用操作 | ❌ 不推荐 | `user + friend` 添加好友 | 运算符不应有副作用 |

**选择决策树**：

```
是否是数学/值类型？
├─ 是 → 运算符语义是否符合直觉？
│   ├─ 是 → ✅ 使用运算符重载
│   └─ 否 → ❌ 使用方法
└─ 否 → 是否是集合索引访问？
    ├─ 是 → ✅ 使用 [] 和 []=
    └─ 否 → ❌ 使用方法(更清晰)
```

---

## 5. 可调用类（Callable Classes）

### 5.1 可调用类概念

**什么是可调用类？**

可调用类（Callable Classes）是指实现了 `call()` 方法的类。这种类的实例可以像函数一样被调用，提供了一种优雅的方式来封装带状态的函数行为。

**为什么需要可调用类？**

传统的函数无法携带状态，而可调用类可以：
- ✅ 封装配置和状态
- ✅ 提供类似函数的调用语法
- ✅ 支持面向对象的特性（继承、多态等）
- ✅ 更好的代码组织和复用

**基本概念**：

```dart
// 普通函数:无状态
int add(int a, int b) => a + b;

// 可调用类:可以携带状态
class Adder {
  final int base;
  Adder(this.base);

  // 实现 call 方法
  int call(int value) => base + value;
}

void main() {
  final add5 = Adder(5);
  print(add5(3)); // 8 - 像函数一样调用
  print(add5.call(3)); // 8 - 显式调用 call 方法
}
```

---

### 5.2 call 方法语法

**基本语法**：

```dart
class ClassName {
  // call 方法可以有任意参数和返回值
  ReturnType call(参数列表) {
    // 实现
  }
}
```

**语法特点**：
- ✅ `call` 是一个特殊的方法名
- ✅ 可以有任意数量和类型的参数（位置参数、可选参数、命名参数）
- ✅ 可以有任意返回类型（包括 void）
- ✅ 可以是泛型方法
- ⚠️ 一个类只能有一个 `call` 方法（不支持重载）

#### 示例：可调用计数器

```dart
/// 可调用计数器：每次调用累加指定步长
class Counter {
  int _count = 0;

  /// call 方法支持可选参数
  int call([int step = 1]) {
    _count += step;
    return _count;
  }

  int get current => _count;
}

void main() {
  final counter = Counter();

  print(counter()); // 1 - 默认步长为1
  print(counter(2)); // 3 - 累加2
  print(counter(5)); // 8 - 累加5
  print('当前值: ${counter.current}'); // 8
}
```

---

### 5.3 与 Function 类型的关系

**可调用类与 Function 类型兼容**：

实现了 `call` 方法的类实例可以赋值给对应签名的 `Function` 类型变量。

```dart
class Multiplier {
  final int factor;
  Multiplier(this.factor);

  int call(int value) => value * factor;
}

void main() {
  final times3 = Multiplier(3);

  // 可调用对象可以赋值给 Function 类型
  int Function(int) func = times3;
  print(func(5)); // 15

  // 也可以作为函数参数传递
  final numbers = [1, 2, 3, 4];
  final result = numbers.map(times3).toList();
  print(result); // [3, 6, 9, 12]
}
```

**关键要点**：
- ✅ 可调用对象可以赋值给匹配签名的 `Function` 类型
- ✅ 可以作为高阶函数的参数传递
- ✅ 类型检查在编译时进行
- 📌 `call` 方法的签名必须与 `Function` 类型匹配

---

### 5.4 实际应用场景

#### 场景1：带配置的验证器

可调用类非常适合封装带配置的验证逻辑：

```dart
/// 带配置的密码验证器
class PasswordValidator {
  final int minLength;
  final bool requireNumber;
  final bool requireUppercase;
  final RegExp _digit = RegExp(r'\d');
  final RegExp _upper = RegExp(r'[A-Z]');

  PasswordValidator({
    this.minLength = 8,
    this.requireNumber = true,
    this.requireUppercase = false,
  });

  /// call 方法执行验证，返回错误信息或 null
  String? call(String password) {
    if (password.length < minLength) {
      return '密码长度至少 $minLength 位';
    }
    if (requireNumber && !_digit.hasMatch(password)) {
      return '密码必须包含数字';
    }
    if (requireUppercase && !_upper.hasMatch(password)) {
      return '密码必须包含大写字母';
    }
    return null; // 验证通过
  }
}

void main() {
  // 创建不同配置的验证器
  final basicValidator = PasswordValidator(
    minLength: 6,
    requireUppercase: true,
  );
  final strictValidator = PasswordValidator(
    minLength: 10,
    requireNumber: true,
    requireUppercase: true,
  );

  // 像函数一样使用
  print(basicValidator('Pass12')); // null（通过）
  print(basicValidator('pass12')); // 密码必须包含大写字母
  print(strictValidator('Pass12')); // 密码长度至少 10 位
}
```

#### 场景2：策略模式实现

可调用类可以优雅地实现策略模式：

```dart
/// 排序策略基类
abstract class SortStrategy {
  List<int> call(List<int> data);
}

/// 升序排序策略
class AscendingSort implements SortStrategy {
  @override
  List<int> call(List<int> data) {
    final copy = List<int>.from(data);
    copy.sort();
    return copy;
  }
}

/// 降序排序策略
class DescendingSort implements SortStrategy {
  @override
  List<int> call(List<int> data) {
    final copy = List<int>.from(data);
    copy.sort((a, b) => b.compareTo(a));
    return copy;
  }
}

void main() {
  final numbers = [3, 1, 4, 1, 5, 9, 2, 6];

  // 使用不同策略
  final ascending = AscendingSort();
  final descending = DescendingSort();

  print('升序: ${ascending(numbers)}'); // [1, 1, 2, 3, 4, 5, 6, 9]
  print('降序: ${descending(numbers)}'); // [9, 6, 5, 4, 3, 2, 1, 1]
}
```

---

### 5.5 常见陷阱与最佳实践

#### 陷阱1：可变状态导致的并发问题

```dart
// ❌ 错误：可变状态可能导致意外行为
class BadCounter {
  int count = 0;

  int call() => ++count; // 每次调用都修改状态
}

void main() {
  final counter = BadCounter();
  final results = [1, 2, 3].map((_) => counter()).toList();
  print(results); // [1, 2, 3] - 依赖调用顺序
}

// ✅ 正确：使用不可变状态或明确状态管理
class GoodCounter {
  final int _initial;
  GoodCounter(this._initial);

  int call(int increment) => _initial + increment; // 无副作用
}
```

**最佳实践**：
- ✅ 优先使用不可变状态
- ✅ 如果需要可变状态，明确文档说明
- ⚠️ 注意多线程环境下的状态安全

---

### 5.6 可调用类 vs typedef vs 函数对象

| 特性 | 可调用类 | typedef | 函数对象 |
|------|---------|---------|---------|
| 携带状态 | ✅ 可以 | ❌ 不可以 | ✅ 通过闭包捕获 |
| 类型别名 | ❌ 需要完整类名 | ✅ 提供简洁别名 | ❌ 匿名 |
| 面向对象特性 | ✅ 支持继承/多态 | ❌ 仅类型定义 | ❌ 无 |
| 配置灵活性 | ✅ 构造函数配置 | ❌ 无 | ⚠️ 依赖闭包捕获 |
| 代码可读性 | ✅ 清晰的类结构 | ✅ 简洁的类型 | ⚠️ 可能不清晰 |
| 适用场景 | 带配置的复杂逻辑 | 函数类型定义 | 简单闭包 |

**选择建议**：

```
需要携带配置或状态？
├─ 是 → 需要继承/多态？
│   ├─ 是 → ✅ 使用可调用类
│   └─ 否 → ✅ 使用可调用类或闭包
└─ 否 → 需要类型别名？
    ├─ 是 → ✅ 使用 typedef
    └─ 否 → ✅ 使用普通函数或闭包
```

**关键要点**：
- ✅ 可调用类适合封装带状态的函数行为
- ✅ typedef 适合定义函数类型别名
- ✅ 简单场景优先使用普通函数或闭包
- 📌 可调用类提供了函数式编程和面向对象编程的桥梁

---

## 6. Records（记录类型）

### 6.1 核心概念

Records（记录类型）在 Dart 3 中提供轻量、不可变的聚合数据结构，用于同时返回多个值而无需声明类。它拥有以下特性：

1. **结构化等价**：位置字段和命名字段决定相等性与 `hashCode`。
2. **模式友好**：与 Pattern Matching 深度集成，可直接解构。
3. **零额外开销**：运行时表现接近普通对象，支持 `const`。

### 6.2 语法速查

- 仅位置字段：`(String, int) hero = ('Dart', 3);`
- 混合命名字段：`({String name, int age}) user = (name: 'Bob', age: 18);`
- 解构：`var (:name, :age) = user;`
- 方法返回记录：`({User user, Settings settings}) load()`。

```dart
typedef AuthResult = ({String token, DateTime expiresAt});

AuthResult login(String user, String password) {
  // 业务逻辑……
  return (token: 'jwt_token', expiresAt: DateTime.now().add(const Duration(hours: 2)));
}

void main() {
  final (:token, :expiresAt) = login('alice', 'p@ss');
  print('Token: $token, 过期时间: $expiresAt');
}
```

### 6.3 Flutter 实战：聚合接口返回多实体

```dart
import 'package:flutter/material.dart';

class UserProfile {
  final String name;
  final String avatar;
  const UserProfile(this.name, this.avatar);
}

class Preferences {
  final ThemeMode themeMode;
  final Locale locale;
  const Preferences(this.themeMode, this.locale);
}

typedef ProfileRecord = ({UserProfile profile, Preferences prefs});

Future<ProfileRecord> loadProfile() async {
  // 假装调用多个后端接口
  await Future.delayed(const Duration(milliseconds: 200));
  return (
    profile: const UserProfile('Alice', 'https://example.com/avatar.png'),
    prefs: const Preferences(ThemeMode.dark, Locale('zh'))
  );
}

class ProfileScreen extends StatelessWidget {
  const ProfileScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<ProfileRecord>(
      future: loadProfile(),
      builder: (context, snapshot) {
        return switch (snapshot) {
          AsyncSnapshot(hasData: true, data: final data) => _ProfileView(data),
          AsyncSnapshot(hasError: true) => const Center(child: Text('加载失败')),
          _ => const Center(child: CircularProgressIndicator()),
        };
      },
    );
  }
}

class _ProfileView extends StatelessWidget {
  const _ProfileView(this.data);
  final ProfileRecord data;

  @override
  Widget build(BuildContext context) {
    final (:profile, :prefs) = data;
    return ListTile(
      leading: CircleAvatar(backgroundImage: NetworkImage(profile.avatar)),
      title: Text(profile.name),
      subtitle: Text('主题: ${prefs.themeMode.name}, 语言: ${prefs.locale.languageCode}'),
    );
  }
}
```

**关键要点**：
- 把多实体组装成一个记录，省去 DTO/数据类。
- Flutter 组件中使用 `(:field, :)` 解构直接拿到字段，类型推断清晰。
- 结合 `FutureBuilder`/`switch`，可读性与类型安全兼备。

### 6.4 常见陷阱与最佳实践

- ⚠️ 记录是不可变结构，字段不能重新赋值。
- ⚠️ 命名字段顺序不影响等价性，但位置字段顺序不同即不等价。
- ✅ 使用 `typedef` 为复杂记录命名，便于复用。
- ✅ Flutter 中推荐与 `pattern` Switch 结合，构建更声明式的 UI。

---

## 7. Pattern Matching 与解构

### 7.1 语法概览

Pattern Matching 允许对对象、记录、集合做结构化匹配。常见语法：

- `switch` 表达式：`switch (value) { pattern => result, _ => ... }`
- `if`/`while` pattern：`if (user case User(isActive: true)) { … }`
- 解构赋值：`var (a, b) = tuple;`
- 守卫：`case Command(:final route) when route.startsWith('/admin')`.

### 7.2 结合记录的匹配

```dart
({String type, int value}) command = (type: 'resize', value: 200);

switch (command) {
  (:final type, :final value) when type == 'resize' => print('调整为 $value'),
  _ => print('未知命令'),
}
```

### 7.3 Flutter 实战：根据网络状态渲染

```dart
import 'package:flutter/material.dart';

sealed class NetworkState {
  const NetworkState();
}

class NetworkLoading extends NetworkState {
  const NetworkLoading();
}

class NetworkSuccess extends NetworkState {
  const NetworkSuccess(this.data);
  final String data;
}

class NetworkError extends NetworkState {
  const NetworkError(this.message);
  final String message;
}

Widget buildNetworkBody(NetworkState state) {
  return switch (state) {
    NetworkLoading() => const CircularProgressIndicator(),
    NetworkSuccess(:final data) => Text(data),
    NetworkError(:final message) => Text(message, style: const TextStyle(color: Colors.red)),
  };
}
```

在 `FutureBuilder` 中：

```dart
Widget build(BuildContext context) {
  return FutureBuilder<String>(
    future: fetch(),
    builder: (context, snapshot) {
      final NetworkState state = switch (snapshot) {
        AsyncSnapshot(connectionState: ConnectionState.waiting) => const NetworkLoading(),
        AsyncSnapshot(hasError: true, error: final error) => NetworkError('$error'),
        AsyncSnapshot(hasData: true, data: final data) => NetworkSuccess(data),
        _ => const NetworkLoading(),
      };
      return buildNetworkBody(state);
    },
  );
}
```

### 7.4 注意事项

- ✅ 使用 `sealed` / `enhanced enum` 可获得穷尽检查。
- ⚠️ Pattern 匹配依赖静态类型，`dynamic` 会跳过编译期校验。
- ✅ 在 Flutter 中可将 `AsyncSnapshot`、`RouteSettings` 等常见类型封装为 pattern，减少 `if` 链。

---

## 8. Sealed Classes & Enhanced Enums

### 8.1 概念速览

- `sealed`：限制子类只能在同一 library 内定义，便于穷尽匹配。
- Enhanced Enums：枚举可拥有字段、方法、接口，实现更强表达力。

### 8.2 Flutter 状态机示例

```dart
import 'package:flutter/material.dart';

sealed class CheckoutState {
  const CheckoutState();
}

class CheckoutIdle extends CheckoutState {
  const CheckoutIdle();
}

class CheckoutProcessing extends CheckoutState {
  const CheckoutProcessing(this.progress);
  final double progress;
}

class CheckoutFailed extends CheckoutState {
  const CheckoutFailed(this.message);
  final String message;
}

class CheckoutSucceeded extends CheckoutState {
  const CheckoutSucceeded();
}

class CheckoutPage extends StatelessWidget {
  const CheckoutPage({super.key, required this.state});
  final CheckoutState state;

  @override
  Widget build(BuildContext context) {
    return switch (state) {
      CheckoutIdle() => ElevatedButton(onPressed: () {}, child: const Text('开始支付')),
      CheckoutProcessing(:final progress) => LinearProgressIndicator(value: progress),
      CheckoutFailed(:final message) => Text(message, style: const TextStyle(color: Colors.red)),
      CheckoutSucceeded() => const Icon(Icons.check_circle, color: Colors.green, size: 48),
    };
  }
}
```

### 8.3 增强枚举驱动 UI

```dart
enum SupportChannel {
  email('邮件', Icons.email, Colors.blue),
  chat('实时聊天', Icons.chat, Colors.green),
  phone('电话', Icons.phone, Colors.orange);

  const SupportChannel(this.label, this.icon, this.color);
  final String label;
  final IconData icon;
  final Color color;

  Widget buildAction(VoidCallback onTap) {
    return ElevatedButton.icon(
      onPressed: onTap,
      icon: Icon(icon),
      label: Text(label),
      style: ElevatedButton.styleFrom(backgroundColor: color),
    );
  }
}
```

### 8.4 最佳实践

- ✅ 将 `Bloc`/`Notifier` 状态设计为 `sealed`，Widget `switch` 时可获得编译期"穷尽"提醒。
- ✅ 增强枚举可承载图标、颜色等 UI 元信息，消除 `switch` 嵌套。
- ⚠️ `sealed` 仍可被 `mixin`/`implements`，需限制库边界。
- ⚠️ Enhanced Enum 默认是 `const` 实例，字段需为 `final`。

---

## 9. Extension Types（扩展类型）

### 9.1 核心概念

Extension Types 提供"零成本包装"，允许为现有类型创建额外语义，而不会生成新的运行时对象（值在编译期重用）。典型用途：

- 语义化 ID/值对象（`UserId`, `OrderCode`）。
- 整合第三方库但保持类型安全。
- 将额外行为封装在扩展类型方法中。

### 9.2 语法示例

```dart
extension type const UserId(String value) {
  const UserId(this.value);

  int get asInt => int.parse(value);
  @override
  String toString() => '#$value';
}

extension type const HexColor(String value) implements String {
  const HexColor(this.value);

  Color toColor() => Color(int.parse('0xFF$value'));
}
```

### 9.3 Flutter 实战：单位与颜色安全

```dart
import 'package:flutter/material.dart';

extension type const Px(double value) {
  const Px(this.value);
  double toLogical(BuildContext context) => value / MediaQuery.of(context).devicePixelRatio;
}

class AccentCard extends StatelessWidget {
  const AccentCard({super.key, required this.color, required this.padding});
  final HexColor color;
  final Px padding;

  @override
  Widget build(BuildContext context) {
    return Container(
      color: color.toColor(),
      padding: EdgeInsets.all(padding.toLogical(context)),
      child: Text('Hex: ${color.value}'),
    );
  }
}

void main() {
  const accent = HexColor('FF6B6B');
  runApp(const MaterialApp(
    home: Scaffold(
      body: Center(child: AccentCard(color: accent, padding: Px(32))),
    ),
  ));
}
```

### 9.4 限制与建议

- ⚠️ 实验阶段特性（Dart 3.4+），启用前确认 SDK 版本。
- ⚠️ 不支持存储实例字段，只能使用"表示类型"的字段。
- ✅ 可通过 `implements` 继承接口能力，如上例实现 `String` 行为。
- ✅ Flutter 中可用来区分物理像素/逻辑像素、ID 字符串等，避免 `typedef = String` 带来的混淆。

---

## 10. 综合实践案例

本章通过3个综合案例展示前文高级语法特性的协同使用方式。

### 10.1 案例1：类型安全的状态管理器

**涉及特性**：泛型约束 + Mixin + 扩展方法

**场景描述**：创建一个类型安全的状态管理器，支持状态监听和历史记录功能。

```dart
/// 状态基类：要求拥有时间戳
abstract class ViewState {
  DateTime get timestamp;
}

typedef StateListener<T extends ViewState> = void Function(T state);

/// 提供历史记录的 Mixin
mixin HistoryMixin<T extends ViewState> {
  final List<T> _history = [];

  void record(T state) => _history.add(state);

  List<T> get history => List.unmodifiable(_history);
}

/// 扩展方法：快速获取最近一次状态
extension LatestStateExt<T extends ViewState> on HistoryMixin<T> {
  T? get latest {
    final snapshot = history;
    return snapshot.isNotEmpty ? snapshot.last : null;
  }
}

/// 类型安全的状态管理器：支持监听与历史
class StateManager<T extends ViewState> with HistoryMixin<T> {
  final List<StateListener<T>> _listeners = [];
  late T _state;

  StateManager(T initialState) {
    update(initialState);
  }

  T get state => _state;

  void addListener(StateListener<T> listener) => _listeners.add(listener);

  void removeListener(StateListener<T> listener) => _listeners.remove(listener);

  void update(T newState) {
    _state = newState;
    record(newState);
    for (final listener in _listeners) {
      listener(newState);
    }
  }
}

/// 具体状态实现
class UserViewState implements ViewState {
  final String userName;
  final bool isLoggedIn;
  @override
  final DateTime timestamp;

  UserViewState(this.userName, this.isLoggedIn) : timestamp = DateTime.now();

  @override
  String toString() =>
      'UserViewState(userName: $userName, loggedIn: $isLoggedIn, time: $timestamp)';
}

void main() {
  final manager = StateManager<UserViewState>(
    UserViewState('Alice', false),
  );

  manager.addListener((state) => print('监听到状态: $state'));

  manager.update(UserViewState('Alice', true));
  manager.update(UserViewState('Bob', true));

  print('历史记录: ${manager.history}');
  print('最近状态: ${manager.latest}');
}
```

**特性协同说明**：
- ✅ **泛型约束**：`T extends ViewState` 确保类型安全
- ✅ **Mixin**：`HistoryMixin` 提供可复用的历史记录功能
- ✅ **扩展方法**：`LatestStateExt` 为 Mixin 添加便捷方法
- 📌 三者结合实现了灵活且类型安全的状态管理

---

### 10.2 案例2：DSL 构建器

**涉及特性**：可调用类 + 运算符重载

**场景描述**：创建一个流畅的DSL用于构建SQL查询条件。

```dart
/// 代表单个条件的类
class Condition {
  final String expression;
  const Condition(this.expression);

  /// 运算符重载：支持 AND / OR 链式语法
  Condition operator &(Condition other) =>
      Condition('($expression AND ${other.expression})');

  Condition operator |(Condition other) =>
      Condition('($expression OR ${other.expression})');

  @override
  String toString() => expression;
}

/// 字段对象：通过 call 方法构建条件
class Field {
  final String name;
  const Field(this.name);

  /// call 方法：像函数一样调用
  Condition call(String op, Object value) {
    final formatted = value is String ? "'$value'" : value;
    return Condition('$name $op $formatted');
  }
}

/// DSL 构建器：可调用类封装整体查询
class QueryBuilder {
  final List<Condition> _conditions = [];

  void add(Condition condition) => _conditions.add(condition);

  String build() => _conditions.isEmpty
      ? 'SELECT * FROM users;'
      : 'SELECT * FROM users WHERE ${_conditions.join(' AND ')};';

  /// call 方法接受 DSL 回调，返回最终 SQL
  String call(void Function(QueryBuilder builder) block) {
    block(this);
    final result = build();
    _conditions.clear(); // 清空条件，确保每次调用独立
    return result;
  }
}

void main() {
  const name = Field('name');
  const age = Field('age');
  const city = Field('city');

  final builder = QueryBuilder();
  final sql = builder((b) {
    final condition = (age('>=', 18) & city('=', 'Beijing')) | name('=', 'Admin');
    b.add(condition);
  });

  print(sql);
  // 输出：SELECT * FROM users WHERE ((age >= 18 AND city = 'Beijing') OR name = 'Admin');
}
```

**特性协同说明**：
- ✅ **可调用类**：`Field` 和 `QueryBuilder` 都实现了 `call` 方法
- ✅ **运算符重载**：`&` 和 `|` 运算符实现了条件组合
- 📌 两者结合创建了简洁优雅的DSL语法

---

### 10.3 案例3：插件系统设计

**涉及特性**：Mixin + 泛型约束

**场景描述**：设计一个可扩展的插件系统，支持动态注册和卸载插件。

```dart
/// 插件接口：所有插件必须实现
abstract class Plugin {
  String get name;
  void onRegister();
  void onDispose();
}

/// 插件宿主 Mixin，约束 T 必须是 Plugin
mixin PluginHost<T extends Plugin> {
  final Map<String, T> _plugins = {};

  void register(T plugin) {
    if (_plugins.containsKey(plugin.name)) {
      throw StateError('插件 ${plugin.name} 已存在');
    }
    _plugins[plugin.name] = plugin;
    plugin.onRegister();
  }

  void uninstall(String name) {
    final plugin = _plugins.remove(name);
    if (plugin != null) {
      plugin.onDispose();
    }
  }

  Iterable<T> get plugins => _plugins.values;
}

/// 具体插件实现
class AnalyticsPlugin implements Plugin {
  @override
  final String name = 'analytics';

  @override
  void onRegister() => print('Analytics 插件已加载');

  @override
  void onDispose() => print('Analytics 插件已卸载');
}

class PushPlugin implements Plugin {
  @override
  final String name = 'push';

  @override
  void onRegister() => print('Push 插件已加载');

  @override
  void onDispose() => print('Push 插件已卸载');
}

/// 应用宿主，通过 with 复用插件能力
class Application with PluginHost<Plugin> {
  void bootstrap() {
    register(AnalyticsPlugin());
    register(PushPlugin());
  }
}

void main() {
  final app = Application();
  app.bootstrap();

  for (final plugin in app.plugins) {
    print('运行中的插件: ${plugin.name}');
  }

  app.uninstall('analytics');
}
```

**特性协同说明**：
- ✅ **Mixin**：`PluginHost` 提供可复用的插件管理能力
- ✅ **泛型约束**：`T extends Plugin` 确保类型安全
- 📌 两者结合实现了灵活且类型安全的插件系统

---

## 11. 总结与最佳实践

### 11.1 各特性适用场景总结

| 特性 | 最佳适用场景 | 避免使用场景 |
|------|------------|------------|
| **泛型约束** | 需要类型安全的通用组件、集合操作、状态管理 | 简单的单一类型场景 |
| **协变/逆变** | 集合类型转换、回调函数类型、框架设计 | 日常业务代码（容易出错） |
| **Mixin** | 横切关注点、功能复用、多重行为组合 | 简单的单一继承场景 |
| **扩展方法** | 为第三方库添加功能、语法糖、工具方法 | 需要访问私有成员的场景 |
| **运算符重载** | 数学类型、值对象、DSL构建 | 业务逻辑、有副作用的操作 |
| **可调用类** | 带配置的函数行为、策略模式、状态封装 | 简单的无状态函数 |
| **Records** | 聚合多个返回值、跨接口拼装数据、Widget 组合输出 | 需要可变对象的场景 |
| **Pattern Matching** | 状态机、数据解构、Flutter 渲染分支 | 输入类型不确定且需 `dynamic` 的情况 |
| **Sealed Classes / Enhanced Enums** | UI 状态机、流程编排、语义化枚举 | 不需要穷尽匹配的简单枚举 |
| **Extension Types** | 语义化 ID、单位换算、与 Flutter SDK 交互 | 需要存储可变字段或旧 Dart 版本 |

---

### 11.2 特性选择决策树

```
需要代码复用？
├─ 是 → 需要多重继承？
│   ├─ 是 → ✅ 使用 Mixin
│   └─ 否 → 需要为第三方类添加功能？
│       ├─ 是 → ✅ 使用扩展方法
│       └─ 否 → ✅ 使用普通继承
└─ 否 → 需要类型安全的通用代码？
    ├─ 是 → ✅ 使用泛型约束
    └─ 否 → 需要自定义运算符？
        ├─ 是 → 是数学/值类型？
        │   ├─ 是 → ✅ 使用运算符重载
        │   └─ 否 → ❌ 使用普通方法
        └─ 否 → 需要携带状态的函数？
            ├─ 是 → ✅ 使用可调用类
            └─ 否 → ✅ 使用普通函数
```

---

### 11.3 性能考虑

**泛型与运行时性能**：
- ✅ Dart 的泛型是具体化的（reified），运行时保留类型信息
- ✅ 泛型约束不会带来额外的运行时开销
- 📌 类型检查在编译时完成，运行时性能优秀

**Mixin 的性能影响**：
- ✅ Mixin 在编译时展开，运行时无额外开销
- ✅ 方法调用性能与普通继承相同
- ⚠️ 过多的 Mixin 会增加类的大小

**扩展方法的性能**：
- ✅ 扩展方法是静态解析的，性能优秀
- ✅ 编译后与普通静态方法调用相同
- 📌 不支持多态，但性能更好

**运算符重载的性能**：
- ✅ 运算符重载编译为普通方法调用
- ✅ 性能与普通方法相同
- 📌 不会带来额外开销

**可调用类的性能**：
- ✅ call 方法调用性能与普通方法相同
- ⚠️ 如果携带大量状态，可能影响内存占用
- 📌 合理设计状态结构可以优化性能

---

### 11.4 代码可维护性建议

**1. 适度使用高级特性**：
- ✅ 优先使用简单直接的解决方案
- ⚠️ 不要为了使用高级特性而使用
- 📌 代码的可读性比炫技更重要

**2. 保持一致性**：
- ✅ 在项目中统一使用风格
- ✅ 建立团队编码规范
- 📌 一致性比个人偏好更重要

**3. 文档和注释**：
- ✅ 为复杂的泛型约束添加注释
- ✅ 说明 Mixin 的使用目的
- ✅ 记录运算符重载的语义
- 📌 让代码自解释，注释补充意图

**4. 测试覆盖**：
- ✅ 为泛型代码编写类型安全测试
- ✅ 测试 Mixin 的组合行为
- ✅ 验证运算符重载的边界情况
- 📌 高级特性更需要充分测试

**5. 渐进式采用**：
- ✅ 从简单场景开始使用
- ✅ 逐步积累经验
- ✅ 在团队中分享最佳实践
- 📌 不要一次性重构整个项目

---

## 结语

本文档介绍了 Dart/Flutter 的9个核心高级语法特性：泛型约束与协变/逆变、Mixin 多继承机制、扩展方法、运算符重载、可调用类、Records、Pattern Matching、Sealed Classes & Enhanced Enums 以及 Extension Types。这些特性为开发者提供了强大的工具，但也需要谨慎使用。

**关键要点**：
- ✅ 理解每个特性的适用场景和限制
- ✅ 优先考虑代码的可读性和可维护性
- ✅ 在实践中积累经验，形成最佳实践
- 📌 高级特性是工具，不是目的

希望本文档能帮助您更好地掌握这些高级语法特性，编写出更优雅、更高效的 Dart/Flutter 代码。

---

**文档信息**：
- 版本：v1.0
- 基于：Dart 3.3 / Flutter 3.19+
- 最后更新：2026-01-05
- 作者：Claude Sonnet 4.5
