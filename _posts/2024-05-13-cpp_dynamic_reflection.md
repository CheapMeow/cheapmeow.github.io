---
title: 'C++动态反射学习'
show_date: true
permalink: /posts/2024/05/cpp_dynamic_reflection/
tags:
  - Cpp
  - Reflection
  - Template programming
toc: true
toc_sticky: true
author_profile: false
---

## C++动态反射学习

### Piccolo 中的动态反射

#### 存储类型信息

反射需要存储一些类型信息，Piccolo 用一个 map 存储了类型名对应的工具函数，也就是存储了类型信息。

```cpp
// reflection.h

typedef std::function<void(void*, void*)>      SetFuncion;
typedef std::function<void*(void*)>            GetFuncion;
typedef std::function<const char*()>           GetNameFuncion;
typedef std::function<void(int, void*, void*)> SetArrayFunc;
typedef std::function<void*(int, void*)>       GetArrayFunc;
typedef std::function<int(void*)>              GetSizeFunc;
typedef std::function<bool()>                  GetBoolFunc;
typedef std::function<void(void*)>             InvokeFunction;

typedef std::function<void*(const Json&)>                           ConstructorWithJson;
typedef std::function<Json(void*)>                                  WriteJsonByName;
typedef std::function<int(Reflection::ReflectionInstance*&, void*)> GetBaseClassReflectionInstanceListFunc;

typedef std::tuple<SetFuncion, GetFuncion, GetNameFuncion, GetNameFuncion, GetNameFuncion, GetBoolFunc>
                                                    FieldFunctionTuple;
typedef std::tuple<GetNameFuncion, InvokeFunction> MethodFunctionTuple;
typedef std::tuple<GetBaseClassReflectionInstanceListFunc, ConstructorWithJson, WriteJsonByName> ClassFunctionTuple;
typedef std::tuple<SetArrayFunc, GetArrayFunc, GetSizeFunc, GetNameFuncion, GetNameFuncion>      ArrayFunctionTuple;

// reflection.cpp

static std::map<std::string, ClassFunctionTuple*>       m_class_map;
static std::multimap<std::string, FieldFunctionTuple*>  m_field_map;
static std::multimap<std::string, MethodFunctionTuple*> m_method_map;
static std::map<std::string, ArrayFunctionTuple*>       m_array_map;
```

这些工具函数就是用来访问反射实例的。

工具函数是怎么来的？在业务代码的定义里面添加宏定义，之后分析业务代码，生成与需要反射的业务代码中的类型一一对应的反射工具类，其中定义了这些工具函数。

这些生成的反射工具类中包含一组组的工具函数。你需要什么反射功能，你就需要规定生成什么样的工具函数。现在已经写得明确了，就是四组工具函数，`FieldFunctionTuple` 用来访问字段；`MethodFunctionTuple` 用来访问方法；`ClassFunctionTuple` 用来获取基类列表、从 json 反序列化为类的实例、从类的实例序列化为 json；`ArrayFunctionTuple` 用来访问数组

```cpp
// vector2.reflection.gen.h

namespace Piccolo{
    class Vector2;
namespace Reflection{
namespace TypeFieldReflectionOparator{
    class TypeVector2Operator{
    public:
        static const char* getClassName(){ return "Vector2";}
        static void* constructorWithJson(const Json& json_context){
            Vector2* ret_instance= new Vector2;
            Serializer::read(json_context, *ret_instance);
            return ret_instance;
        }
        static Json writeByName(void* instance){
            return Serializer::write(*(Vector2*)instance);
        }
        // base class
        static int getVector2BaseClassReflectionInstanceList(ReflectionInstance* &out_list, void* instance){
            int count = 0;
            
            return count;
        }
        // fields
        static const char* getFieldName_x(){ return "x";}
        static const char* getFieldTypeName_x(){ return "float";}
        static void set_x(void* instance, void* field_value){ static_cast<Vector2*>(instance)->x = *static_cast<float*>(field_value);}
        static void* get_x(void* instance){ return static_cast<void*>(&(static_cast<Vector2*>(instance)->x));}
        static bool isArray_x(){ return false; }
        static const char* getFieldName_y(){ return "y";}
        static const char* getFieldTypeName_y(){ return "float";}
        static void set_y(void* instance, void* field_value){ static_cast<Vector2*>(instance)->y = *static_cast<float*>(field_value);}
        static void* get_y(void* instance){ return static_cast<void*>(&(static_cast<Vector2*>(instance)->y));}
        static bool isArray_y(){ return false; }

        // methods
        
    };
}//namespace TypeFieldReflectionOparator
```

他这里为了不污染业务代码，把反射生成的工具函数和工具类都放在了一个独立的命名空间里面。每一组工具函数也是处于不同的命名空间里面，看上去更清晰，这里只是展示了字段的工具函数，位于 `TypeFieldReflectionOparator`，如果原始类型需要反射数组，相关的工具函数就会放在另外的命名空间 `ArrayReflectionOperator` 中

工具函数这里可以看到，输入的 instance 是 `void*`。

```cpp
// vector2.reflection.gen.h

static void* get_x(void* instance){ return static_cast<void*>(&(static_cast<Vector2*>(instance)->x));}
```

这其实隐含着一个约定是，反射实例的数据都用 `void*` 表示。`void*` 很常用，因为使用指针可以避免数据拷贝，使用 `void` 是方便 cast。

如果之后和别的反射框架对照，可以发现，你为了统一处理反射逻辑，肯定会把需要反射的东西装箱成一个中间类，或者说，类型擦除

这个中间类你用 `std::any` 存指针或者 `void*` 或者自己定义一个类似 `std::any` 的东西都可以

最终反正都是调用 `static_cast` 把擦除之后的类转换成自定义类型，然后访问自定义类型中的成员，然后再把访问结果用 `static_cast` 类型擦除

它这里为了把表示数据的 `void*` 和表示类型的 `string` 绑在一起，就创建了 `ReflectionInstance` `ReflectionPtr`

```cpp
// reflection.h

class ReflectionInstance
{
    ...

public:
    TypeMeta m_meta;
    void*    m_instance;
};

template<typename T>
class ReflectionPtr
{
    ...
    
private:
    std::string m_type_name {""};
    typedef T   m_type;
    T*          m_instance {nullptr};
};
```

业务代码的反射宏定义中已经把这些反射工具类前向声明为了友元，所以这些反射工具类里面的工具函数是有权限访问到原始类型的成员的

```cpp
// reflection.h

#define REFLECTION_BODY(class_name) \
    friend class Reflection::TypeFieldReflectionOparator::Type##class_name##Operator; \
    friend class Serializer;
    // public: virtual std::string getTypeName() override {return #class_name;}

#define REFLECTION_TYPE(class_name) \
    namespace Reflection \
    { \
        namespace TypeFieldReflectionOparator \
        { \
            class Type##class_name##Operator; \
        } \
    };

// vector2.h

REFLECTION_TYPE(Vector2)
CLASS(Vector2, Fields)
{
    REFLECTION_BODY(Vector2);

    ...

}
```

这只是生成了代码，找了一个位置定义了工具函数，如果我们需要从任意一个位置通过类型名字获取工具函数，我们还需要把这些工具函数统一存储在一个地方，方便查找。

生成的反射工具类中有一个函数，收集四组工具函数，每一组打包成一个 `std::tuple`，加入全局反射单例的 map

```cpp
// animation_clip.reflection.gen.h

void TypeWrapperRegister_AnimNodeMap(){
    FieldFunctionTuple* field_function_tuple_convert=new FieldFunctionTuple(
        &TypeFieldReflectionOparator::TypeAnimNodeMapOperator::set_convert,
        &TypeFieldReflectionOparator::TypeAnimNodeMapOperator::get_convert,
        &TypeFieldReflectionOparator::TypeAnimNodeMapOperator::getClassName,
        &TypeFieldReflectionOparator::TypeAnimNodeMapOperator::getFieldName_convert,
        &TypeFieldReflectionOparator::TypeAnimNodeMapOperator::getFieldTypeName_convert,
        &TypeFieldReflectionOparator::TypeAnimNodeMapOperator::isArray_convert);
    REGISTER_FIELD_TO_MAP("AnimNodeMap", field_function_tuple_convert);

    
    
    ArrayFunctionTuple* array_tuple_stdSSvectorLstdSSstringR = new  ArrayFunctionTuple(
        &ArrayReflectionOperator::ArraystdSSvectorLstdSSstringROperator::set,
        &ArrayReflectionOperator::ArraystdSSvectorLstdSSstringROperator::get,
        &ArrayReflectionOperator::ArraystdSSvectorLstdSSstringROperator::getSize,
        &ArrayReflectionOperator::ArraystdSSvectorLstdSSstringROperator::getArrayTypeName,
        &ArrayReflectionOperator::ArraystdSSvectorLstdSSstringROperator::getElementTypeName);
    REGISTER_ARRAY_TO_MAP("std::vector<std::string>", array_tuple_stdSSvectorLstdSSstringR);
    ClassFunctionTuple* class_function_tuple_AnimNodeMap=new ClassFunctionTuple(
        &TypeFieldReflectionOparator::TypeAnimNodeMapOperator::getAnimNodeMapBaseClassReflectionInstanceList,
        &TypeFieldReflectionOparator::TypeAnimNodeMapOperator::constructorWithJson,
        &TypeFieldReflectionOparator::TypeAnimNodeMapOperator::writeByName);
    REGISTER_BASE_CLASS_TO_MAP("AnimNodeMap", class_function_tuple_AnimNodeMap);
}
```

他这个解析代码的程序生成反射代码文件与源码中的头文件一一对应，源码里面一个头文件可以定义多个需要反射的类，这个程序都可以处理

这种情况下，一个反射工具类对应一个用于收集工具函数，创建 tuple 的函数 `TypeWrapperRegister_ClassName`。一个文件里面最终还需要一个注册函数 `TypeWrappersRegister::ClassName`，里面调用所有的 `TypeWrapperRegister_ClassName`

```cpp
namespace TypeWrappersRegister{
    void AnimationClip()
    {
        TypeWrapperRegister_AnimNodeMap();
        TypeWrapperRegister_AnimationChannel();
        TypeWrapperRegister_AnimationClip();
        TypeWrapperRegister_AnimationAsset();
    }
}//namespace TypeWrappersRegister
```

代码生成程序除了一个需要反射的头文件对应一个反射代码文件之外，还会生成一个专门用于注册类型信息的文件，里面调用了所有的注册函数 `TypeWrappersRegister::ClassName`

```cpp
namespace Reflection{
    void TypeMetaRegister::metaRegister(){
        TypeWrappersRegister::BlendState();
        TypeWrappersRegister::Quaternion();
        TypeWrappersRegister::AxisAligned();
        TypeWrappersRegister::TransformComponent();
        TypeWrappersRegister::ParticleComponent();
        TypeWrappersRegister::Vector3();
        TypeWrappersRegister::Transform();
        TypeWrappersRegister::Color();
        TypeWrappersRegister::Vector4();
        TypeWrappersRegister::Vector2();
        TypeWrappersRegister::Object();
        TypeWrappersRegister::Matrix4();
        TypeWrappersRegister::MetaExample();
        TypeWrappersRegister::World();
        TypeWrappersRegister::AnimationClip();
        TypeWrappersRegister::AnimationSkeletonNodeMap();
        TypeWrappersRegister::SkeletonData();
        TypeWrappersRegister::SkeletonMask();
        TypeWrappersRegister::Animation();
        TypeWrappersRegister::AnimationComponent();
        TypeWrappersRegister::Camera();
        TypeWrappersRegister::Component();
        TypeWrappersRegister::Material();
        TypeWrappersRegister::CameraComponent();
        TypeWrappersRegister::BasicShape();
        TypeWrappersRegister::RigidBody();
        TypeWrappersRegister::LuaComponent();
        TypeWrappersRegister::Mesh();
        TypeWrappersRegister::RenderObject();
        TypeWrappersRegister::MeshComponent();
        TypeWrappersRegister::Motor();
        TypeWrappersRegister::MotorComponent();
        TypeWrappersRegister::GlobalRendering();
        TypeWrappersRegister::Emitter();
        TypeWrappersRegister::RigidbodyComponent();
        TypeWrappersRegister::GlobalParticle();
        TypeWrappersRegister::MeshData();
        TypeWrappersRegister::CameraConfig();
        TypeWrappersRegister::Level();
    }
}
```

这个总的注册函数 `TypeMetaRegister::metaRegister` 在游戏引擎启动的时候被调用，这样就完成了类型信息的存储

#### 使用反射创建 GUI 树

给定一个反射实例，需要知道他的父类有多少个

```cpp
// editor_ui.cpp

void EditorUI::createClassUI(Reflection::ReflectionInstance& instance)
{
    Reflection::ReflectionInstance* reflection_instance;
    int count = instance.m_meta.getBaseClassReflectionInstanceList(reflection_instance, instance.m_instance);
    for (int index = 0; index < count; index++)
    {
        createClassUI(reflection_instance[index]);
    }
    createLeafNodeUI(instance);

    if (count > 0)
        delete[] reflection_instance;
}
```

他这里是认为 GUI 树的布局跟继承树是一样的

创建某一个节点的 GUI 是大量使用反射的地方。对一个反射实例，获取他所有的字段，如果字段是数组，那么就要遍历这个数组的所有成员，每一个成员又要创建一个反射实例，继续调用 `createClassUI`；如果不是，那么用这个字段创建反射实例，继续调用 `createClassUI`

```cpp
// editor_ui.cpp

void EditorUI::createLeafNodeUI(Reflection::ReflectionInstance& instance)
{
    Reflection::FieldAccessor* fields;
    int                        fields_count = instance.m_meta.getFieldsList(fields);

    for (size_t index = 0; index < fields_count; index++)
    {
        auto field = fields[index];
        if (field.isArrayType())
        {
            Reflection::ArrayAccessor array_accessor;
            if (Reflection::TypeMeta::newArrayAccessorFromName(field.getFieldTypeName(), array_accessor))
            {
                void* field_instance = field.get(instance.m_instance);
                int   array_count    = array_accessor.getSize(field_instance);
                m_editor_ui_creator["TreeNodePush"](
                    std::string(field.getFieldName()) + "[" + std::to_string(array_count) + "]", nullptr);
                auto item_type_meta_item =
                    Reflection::TypeMeta::newMetaFromName(array_accessor.getElementTypeName());
                auto item_ui_creator_iterator = m_editor_ui_creator.find(item_type_meta_item.getTypeName());
                for (int index = 0; index < array_count; index++)
                {
                    if (item_ui_creator_iterator == m_editor_ui_creator.end())
                    {
                        m_editor_ui_creator["TreeNodePush"]("[" + std::to_string(index) + "]", nullptr);
                        auto object_instance = Reflection::ReflectionInstance(
                            Piccolo::Reflection::TypeMeta::newMetaFromName(item_type_meta_item.getTypeName().c_str()),
                            array_accessor.get(index, field_instance));
                        createClassUI(object_instance);
                        m_editor_ui_creator["TreeNodePop"]("[" + std::to_string(index) + "]", nullptr);
                    }
                    else
                    {
                        if (item_ui_creator_iterator == m_editor_ui_creator.end())
                        {
                            continue;
                        }
                        m_editor_ui_creator[item_type_meta_item.getTypeName()](
                            "[" + std::to_string(index) + "]", array_accessor.get(index, field_instance));
                    }
                }
                m_editor_ui_creator["TreeNodePop"](field.getFieldName(), nullptr);
            }
        }
        auto ui_creator_iterator = m_editor_ui_creator.find(field.getFieldTypeName());
        if (ui_creator_iterator == m_editor_ui_creator.end())
        {
            Reflection::TypeMeta field_meta =
                Reflection::TypeMeta::newMetaFromName(field.getFieldTypeName());
            if (field.getTypeMeta(field_meta))
            {
                auto child_instance =
                    Reflection::ReflectionInstance(field_meta, field.get(instance.m_instance));
                m_editor_ui_creator["TreeNodePush"](field_meta.getTypeName(), nullptr);
                createClassUI(child_instance);
                m_editor_ui_creator["TreeNodePop"](field_meta.getTypeName(), nullptr);
            }
            else
            {
                if (ui_creator_iterator == m_editor_ui_creator.end())
                {
                    continue;
                }
                m_editor_ui_creator[field.getFieldTypeName()](field.getFieldName(),
                                                                        field.get(instance.m_instance));
            }
        }
        else
        {
            m_editor_ui_creator[field.getFieldTypeName()](field.getFieldName(),
                                                                    field.get(instance.m_instance));
        }
    }
    delete[] fields;
}
```

可以看出，为什么全局反射静态实例里面的 map 的 key 是类型名，`FieldFunctionTuple` 里面还要放一个获取类型名的函数。因为反射的时候我们并不是直接使用这个 map，而是先去获得字段的名字，然后再根据字段的名字从这个 map 中取东西。

其中 `m_editor_ui_creator` 用来存放 Imgui 创建 GUI 的函数。它不仅包含 `TreeNodePush` 这种 Imgui 自带的函数，更重要的的是包含了引擎里面基本类型的处理，这也是递归创建 GUI 的终止的地方

指向数据的指针会传入 `m_editor_ui_creator`，这就完成了 Imgui 对数据的控制

指向数据的指针是使用反射，遍历某个反射类的所有字段取到的，这就完成了 Imgui 能获取并控制所有被反射的字段

显示 GUI 的起点是，指定的物体的 components

从它们开始包装反射实例 `Reflection::ReflectionInstance` 专门是为了配合

```cpp
// editor_ui.cpp

auto&&                 selected_object_components = selected_object->getComponents();
for (auto component_ptr : selected_object_components)
{
    m_editor_ui_creator["TreeNodePush"](("<" + component_ptr.getTypeName() + ">").c_str(), nullptr);
    auto object_instance = Reflection::ReflectionInstance(
        Piccolo::Reflection::TypeMeta::newMetaFromName(component_ptr.getTypeName().c_str()),
        component_ptr.operator->());
    createClassUI(object_instance);
    m_editor_ui_creator["TreeNodePop"](("<" + component_ptr.getTypeName() + ">").c_str(), nullptr);
}
```

#### 序列化与反序列化

序列化开始的时候，我们是知道我们要序列化或者反序列化某个类的，最终反序列化都是通过资产管理器的接口

```cpp
// asset_manager.h

class AssetManager
{
public:
    template<typename AssetType>
    bool loadAsset(const std::string& asset_url, AssetType& out_asset) const
    {
        // read json file to string
        std::filesystem::path asset_path = getFullPath(asset_url);
        std::ifstream asset_json_file(asset_path);
        if (!asset_json_file)
        {
            LOG_ERROR("open file: {} failed!", asset_path.generic_string());
            return false;
        }

        std::stringstream buffer;
        buffer << asset_json_file.rdbuf();
        std::string asset_json_text(buffer.str());

        // parse to json object and read to runtime res object
        std::string error;
        auto&&      asset_json = Json::parse(asset_json_text, error);
        if (!error.empty())
        {
            LOG_ERROR("parse json file {} failed!", asset_url);
            return false;
        }

        Serializer::read(asset_json, out_asset);
        return true;
    }
```

这里最终会调用 `Serializer::read`。它是一个模板函数，有重载有特化。

如果是派生的自定义子类反序列化，那么会调用代码生成的，模板特化版本的 `Serializer::read`。

递归调用下去，遇到 object 类这种类的话，会比较特殊

```cpp
// object.h

REFLECTION_TYPE(ObjectDefinitionRes)
CLASS(ObjectDefinitionRes, Fields)
{
    REFLECTION_BODY(ObjectDefinitionRes);

public:
    std::vector<Reflection::ReflectionPtr<Component>> m_components;
};
```

它里面含有的 `Reflection::ReflectionPtr<Component>` 这个类型的反序列化不是代码生成的，这是手写的

就，这里会有一些比较绕的事情发生。总的序列化流程我们已经知道了，自定义类型 `A` 的反序列化函数 `Serializer::read` 由代码生成，生成的内容就是对 `A` 的每一个成员继续调用 `Serializer::read`

如果 `A` 的成员都是基本类型，那么就会跳到预先写好的特化到基本类型的 `Serializer::read`；如果 `A` 的成员也有自定义类型 `B` `C`，那么就会跳到代码生成的，特化到 `B` `C` 的 `Serializer::read`

自定义类型中有一些特殊的是 `Reflection::ReflectionPtr<T>` 这个类型，你要反序列化生成的是自定义类型的指针，单纯是特化到自定义类型的 `Serializer::read` 还不够用，还需要多写一个实例化，然后获取指针的这个步骤

```cpp
// serializer.h

template<typename T>
static T*& readPointer(const Json& json_context, T*& instance)
{
    assert(instance == nullptr);
    std::string type_name = json_context["$typeName"].string_value();
    assert(!type_name.empty());
    if ('*' == type_name[0])
    {
        instance = new T;
        read(json_context["$context"], *instance);
    }
    else
    {
        instance = static_cast<T*>(
            Reflection::TypeMeta::newFromNameAndJson(type_name, json_context["$context"]).m_instance);
    }
    return instance;
}

template<typename T>
static T*& read(const Json& json_context, Reflection::ReflectionPtr<T>& instance)
{
    std::string type_name = json_context["$typeName"].string_value();
    instance.setTypeName(type_name);
    return readPointer(json_context, instance.getPtrReference());
}
```

`Reflection::ReflectionPtr<T>` 这里存储的是基类 `T` 的指针，用来实现多态。比如一个 gameobject 可能拥有很多不同的 component，但是它们都继承自 component 基类。

但是我们要生成的是派生类实例，派生类的实例化，可想而知，也是需要读取 json 的。然而读取特定类型的 json 我们是依赖于明确地传入一个自定义类型，才能调用到特化版本的 `Serializer::read`

而现在我们处理 `Reflection::ReflectionPtr` 的时候我们只知道是基类类型 `T`，那就没办法调用到特化版本的 `Serializer::read` 了

所以为了处理 `Reflection::ReflectionPtr` 能够调用到特化版本的 `Serializer::read`，需要代码里面某处明确写明了派生类的类型，然后把这个派生类的类型的参数传给 `Serializer::read` 就能调用到特化版本。这个东西写完，或者说这个函数写完之后，处理 `Reflection::ReflectionPtr` 的时候还需要找得到

这就再次用到了反射。`Reflection::ReflectionPtr` 里面存储派生类的类名，我们根据这个类名通过反射获取到反序列化函数 `Reflection::TypeMeta::newFromNameAndJson`，它里面就是特意调用派生类特化版本的 `Serializer::read`

`readPointer` 除了处理 `Reflection::ReflectionPtr<T>` 还会处理 `T*`。这是因为它们都是表示指针的意义才写在一起的，实际上，`Reflection::ReflectionPtr<T>` 也可以算是一个值类型了。

回到我们的总结，反序列化看上去比较绕的就是，对一个类型，递归调用 `Serializer::read`

1. 如果数据成员是基本类型

    调用特化到基本类型的 `Serializer::read`

2. 如果数据成员是自定义类型 `T`

    调用生成的，特化到 `T` 的 `Serializer::read`

3. 如果数据成员是某个类型的指针 `T*`

    先 `new T` 再调用特化到 `T` 的 `Serializer::read`

4. 如果数据成员是 `Reflection::ReflectionPtr<T>`

    从 `Reflection::ReflectionPtr` 拿到派生类类型信息，根据这个类型信息拿到对应派生类的 `Reflection::TypeMeta::newFromNameAndJson`，其中调用生成的，特化到自定义类型的，并且这个自定义类型是继承自 `T` 的 `Serializer::read`

就是这样，不断递归，最终跳到基本类型就结束了

序列化也是同理，递归调用 `Serializer::write`

1. 如果数据成员是基本类型

    调用特化到基本类型的 `Serializer::write`

2. 如果数据成员是自定义类型 `T`

    调用生成的，特化到 `T` 的 `Serializer::write`

3. 如果数据成员是某个类型的指针 `T*`

    `$typeName` 填 `*`，然后调用特化到 `T` 的 `Serializer::write`

4. 如果数据成员是 `Reflection::ReflectionPtr<T>`

    从 `Reflection::ReflectionPtr` 拿到派生类类型信息，根据这个类型信息拿到对应派生类的 `Reflection::TypeMeta::writeByName`，其中调用生成的，特化到自定义类型的，并且这个自定义类型是继承自 `T` 的 `Serializer::write`

因为 cpp 提供了最基础的静态反射，可以知道一个类型是不是指针，所以 `writePointer` 这里就单单明确是用于 `T*` 的，而不是像 `readPointer` 那样，除了处理 `Reflection::ReflectionPtr<T>` 还会处理 `T*`

```cpp
template<typename T>
static Json writePointer(T* instance)
{
    return Json::object { {"$typeName", Json {"*"}}, {"$context", Serializer::write(*instance)} };
}

template<typename T>
static Json write(const Reflection::ReflectionPtr<T>& instance)
{
    T*          instance_ptr = static_cast<T*>(instance.operator->());
    std::string type_name    = instance.getTypeName();
    return Json::object { {"$typeName", Json(type_name)},
                            {"$context", Reflection::TypeMeta::writeByName(type_name, instance_ptr)} };
}

template<typename T>
static Json write(const T& instance)
{

    if constexpr (std::is_pointer<T>::value)
    {
        return writePointer((T)instance);
    }
    else
    {
        static_assert(always_false<T>, "Serializer::write<T> has not been implemented yet!");
        return Json();
    }
}
```

怎么说呢……这个不对称看上去令人强迫症犯了……

### Taichi Graphics cpp reflection

[https://www.bilibili.com/video/BV1MY4y1c7hU](https://www.bilibili.com/video/BV1MY4y1c7hU)
 
#### 使用 static_assert 以调试

因为在模板编译报错的时候，你是没有运行时的中断的，所以也看不到编译模板中到底发生了什么错，编辑器的报错信息里面也不会说

所以可以使用 `static_assert` 来 debug。比如模板中有一个类型是 `tuple_t`，我们想知道它的类型，那么就

```cpp
static_assert(std::is_same<tuple_t, TestClassType>::value, "Debug info");
```

但是要注意，为了方便 debug，这个模板只能被传入一种类型。这样，你要查看的那个类型也就是确定了的，这样才能跟你 hard code 的类型作比较

如果是多处代码调用了这个模板，发生了不同类型的生成，那这个模板里面你的 `static_assert` 虽然在你期望的某个调用处是通过了，但是在你忽略了的地方就没通过，程序还是编译失败

当然编译器虽然不会告诉你具体为什么编译失败，但还是会告诉谁调用这个模板失败了，这个时候你会找到你忽略的地方……但是如果你没有这个避免不同生成的意识，或许你会下意识忽略掉报错信息给你的答案……是，我犯了这样的错误

#### 简化代码以调试

当我的代码结构比较复杂的时候，我的编译器只会报 `std::bad_any_cast` 但是不会说明具体是怎么出错了

等到我简化了代码之后，才出现了明显的报错

```
'initializing': cannot convert from 'std::array<ArgWrap,2>' to 'std::array<ArgWrap,2> &'
```

表明我是不可以把一个值类型转到引用类型

于是回去看了教程，他是先转成指针再解引用，就能得到引用类型

这就触及到了我的知识盲区……解引用 * 会返回引用类型

以前实现赋值运算符重载的时候，返回值类型是 T& 然后函数体里面结尾是 return *this; 我还没多想

原来解引用还真的是解出了引用……

以下是简化出来的，能给予更多调试信息的代码

```cpp
#include <vector>
#include <any>
#include <array>
#include <type_traits>
#include <utility>
#include <functional>
#include <iostream>
#include <memory>
#include <tuple>
#include <vector>

class ArgWrap
{
public:
    template<typename T>
    ArgWrap(T&& val)
    {
        // for debugging, delete all content
    }
};

class MemberFunction
{
public:
    MemberFunction() = default;

    template<typename C, typename R, typename... Args>
    explicit MemberFunction(R(C::* func)(Args...))
    {
        fn_ = [this, func](std::any obj_args) -> std::any {
            // auto& warpped_args = std::any_cast<std::array<ArgWrap, sizeof...(Args) + 1>>(obj_args); // error, 'initializing': cannot convert from 'std::array<ArgWrap,2>' to 'std::array<ArgWrap,2> &'
            auto& warpped_args = *std::any_cast<std::array<ArgWrap, sizeof...(Args) + 1>*>(obj_args); // right
            // ...
            return std::any{};
            };
    }

    template<typename C, typename... Args>
    std::any Invoke(C& c, Args&&... args)
    {
        std::array<ArgWrap, sizeof...(Args) + 1> args_arr = { ArgWrap {std::reference_wrapper<C>(c)},
                                                                 ArgWrap {std::forward<Args>(args)}... };
        return fn_(&args_arr);
    }

private:

    std::function<std::any(std::any)> fn_{ nullptr };
};

class Foo
{
public:

    // std::unique_ptr will result in compile-time error
    std::shared_ptr<float> MakeFloatPtr(float i) { return std::make_shared<float>(i); }
};

int main() {
    Foo foo;
    MemberFunction mem_fun(&Foo::MakeFloatPtr);

    mem_fun.Invoke(foo, 1.0f);

    return 0;
}
```

#### 输出 typeid(T).name() 以调试

虽然我把 `ArgWrap` 装好了，但是测试不通过

```cpp
auto foo_pass_by_cref = foo_t.GetMemberFunc("PassByConstRef");
foo_pass_by_cref.Invoke(f, std::ref(hello_s));
```

经过 debug 之后发现是参数转换到 tuple，传入函数的时候，Foo 对象能转出来，但是 string 转出来为空

于是我就在装入值的时候检查了类型转换

```cpp
template<typename T>
ArgWrap(T&& val)
{
    // Debug type T
    // static_assert(std::is_same<T, void>::value, "Hoi!");
    m_ref_type = std::is_reference_v<T>;
    m_is_const = std::is_const_v<T>;
    if (m_ref_type == 1)
    {
        m_storage = &val;
    }
    else
    {
        m_storage = val;
    }

    std::cout << "typeid(std::decay_t<T>).name() = " << typeid(std::decay_t<T>).name() << std::endl;
    std::cout << "typeid(T).name() = " << typeid(T).name() << std::endl;

    try
    {
        auto result = std::any_cast<const std::string&>(m_storage);
        std::cout << "can any_cast to const std::string&" << std::endl;
    }
    catch (const std::bad_any_cast& e)
    {
        std::cout << "can not any_cast to const std::string&" << std::endl;
    }
    std::cout << std::endl;
}
```

输出

```
typeid(std::decay_t<T>).name() = class std::reference_wrapper<class std::basic_string<char,struct std::char_traits<char>,class std::allocator<char> > >
typeid(T).name() = class std::reference_wrapper<class std::basic_string<char,struct std::char_traits<char>,class std::allocator<char> > >
can not any_cast to const std::string&
```

这就说明 any_cast 没法处理这个转换……

#### any_cast 的严格

做个测试

```cpp
#include <any>
#include <functional>
#include <iostream>
#include <string>

int main() {
    std::string str1 = "Hello";
    auto        str2 = std::ref(str1);
    auto        str3 = static_cast<const std::string&>(str2); // Ok
    std::cout << "str1 = " << str1 << std::endl;
    std::cout << "str3 = " << str3 << std::endl;

    std::any str4 = std::ref(str1);
    // auto     str5 = std::any_cast<const std::string&>(str4); // Crash, std::bad_any_cast
    // std::cout << "str5 = " << str5 << std::endl;

    auto str6 = std::any_cast<decltype(str2)>(str4);
    auto str7 = static_cast<const std::string&>(str6); // Ok
    std::cout << "str7 = " << str7 << std::endl;
}
```

可以看出，`std::ref` 返回出来的这个类型，`reference_wrapper<T>`，如果我 any 里面放的是 `reference_wrapper<T>`，我是不能直接 `any_cast` 到 `T&` 的

我 `any` 里面放的是什么类型，`any_cast` 出来就必须是什么类型，他不会帮你做 `static_cast`

如果真的有 `static_cast` 的需求，那么需要 `any_cast` 之后自己写

#### 总结

他也是动态反射，只是存储函数信息的时候，用模板元编程来生成那个被存的函数

如果不想用模板，那么用代码生成，也可以生成出被存储的函数，比如 Piccolo

在模板元编程时，因为需要操作类型，所以常用 `std::tuple` 打包类型。比如用这个 `std::tuple` 可以做出包含所有参数的类型的一个 tuple

他最后考虑到 `std::any_cast` 需要一一严格对应的类型转换，所以做了一个 `ArgWarp` 来做类型转换的适配

具体来说，`std::any_cast` 需要 cast 到 `const std::string&` 但是实际 `any` 存储的如果是 `std::string` 的话，`std::any_cast` 就会出错

也就是输入的实参类型 `T1` 和记录的函数的形参的类型 `T2` 之间不匹配，`std::any` 里面存的是 `T1`，但是要 `std::any_cast` 到 `T2`。`std::any_cast` 里面是直接比较 `typeid` 的，所以不会成功，即使你 `static_cast` 从 `T1` 到 `T2` 可以成功

他做的 `ArgWarp` 的适配工作是，假设 `std::any` 存储的类型 `T1` 和 Cast 接口要输出的类型 `T2` 的原始类型，也就是 `std::decay_t` 出来的类型都相同，记为 `RawT`

他就考虑到 `T1` `T2` 分别为 value、const ref、non-const ref 的时候的情况，做了一个 3*3 的转换表，使用 `RawT` 和指针和解引用配合，就能得到正确的 `std::any_cast` 方法，使得我能够利用原本严格的 `std::any_cast` 转换两个不同的但是原始类型相同的类型

但是他基于的假设是 `T1` `T2` 的 `std::decay_t` 出来的类型都相同

实际上别人调用你的反射函数的时候，`T1` `T2` 的 `std::decay_t` 出来的类型还真可能不相同

为什么他会这么用？~~因为 `static_cast` 从 `T1` 到 `T2` 可以成功，这是不违背直觉的~~

因为使用者知道从 `T1` 到 `T2` 的隐式转换是可以成功的

这不局限于 `static_cast`，还有可能是调用了类型转换函数，例子：

```cpp
#include <string>
#include <functional>

int main() {
	std::string str1 = "Hello";
	auto str2 = std::ref(str1);
	std::string& str3(str2);
}
```

你可以通过断点看到，构造 `std::string&` 的时候，调用了 `reference_wrapper<T>` 里面的 `T&` 类型转换函数

所以我觉得更省心智的方法应该是，不要在反射系统里面加入这个 `T1` `T2` 的 `std::decay_t` 出来的类型都相同 的隐性约束

### YKIKO reflection

重新看了别人的文章

[https://zhuanlan.zhihu.com/p/670191053](https://zhuanlan.zhihu.com/p/670191053)

他没有写 registry 和 builder 模式，而是把类型信息存在了 Any 类的偏特化模板中

这让我想到，为什么我在包装参数的时候用 `std::any`

为了避免 `std::any` 的限制，我尝试了使用 `void*` 来存

```cpp
class UnsafeAny
{
public:
    template<typename InputClass>
    UnsafeAny(InputClass&& val)
    {
        m_ref_type = std::is_reference_v<InputClass>;
        if (m_ref_type == 1)
        {
            m_storage = &val;
        }
        else
        {
            m_storage = new std::decay_t<InputClass>(val);
        }
    }

    ~UnsafeAny()
    {
        if (!m_ref_type)
        {
            delete m_storage;
        }
    }

    template<typename OutputClass>
    OutputClass Cast()
    {
        using RawTptr = std::decay_t<OutputClass>*;
        return *static_cast<RawTptr>(m_storage);
    }

private:
    void* m_storage {};
    int   m_ref_type {0};
};
```

但是传入 `reference_wrapper<T>` 的时候还是有问题

之前我是以为从 `reference_wrapper<T>` 到 `T` 的转换是隐式调用了 `static_cast`，实际上不是，实际上是调用了类型转换函数

`reference_wrapper<T>` 到 `T` 用 `static_cast` 是转不了的，它们在类型上是没有关系的

你需要先从 `reference_wrapper<T>` 里面取出指针出来

但是这适用不到任意类型，比如

```cpp
class A {
public:
	int x = 1;
};

class B {
public:
	operator int() {
		return 1;
	}

	operator A() {
		return A();
	}
};
```

你完全不知道他能这么类型转换

或许你在装箱的时候可以尝试把输入 cast 到一些确定的类型

但是这样总归是一个需要维护的地方，位于转换表中的东西可以从接口的角度来看可以隐式转换，不在表里面的就不行，行为不统一，还要维护，不如不搞

#### 总结

现在我完全理解了，如果想要使用这样的反射，还真的就要接受这个规则，就是接受这个无法做隐式类型转换的规则

只是在写代码的时候多写 cast 而已，可以接受

那么大的自由度也没必要

<script src="https://utteranc.es/client.js"
        repo="CheapMeow/cheapmeow.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>