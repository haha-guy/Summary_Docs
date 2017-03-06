##Android智能指针
###概述
1. 为了解决循环引用的问题，引入的智能指针计数，这种引用计数技术将对象的引用计数分为强引用计数和弱引用计数两种，其中对象的生命期只受强引用计数控制。在该技术中，一般将有关联的对象划分为"父－子"和"子－父"关系。在“父－子”关系中，父对象通过强引用计数来引用“子”对象；“子－父”关系中，“子”对象通过弱引用计数来引用“父”对象。

2. 无论是轻量级指针，还是强指针或弱指针，实现原理类似，都是**需要对象提供引用计数器，但由智能指针来负责维护这个引用计数器**.

3. Android系统将引用计数器定义为一个公共类，所有支持使用智能指针的对象类都必须从这个公共类继承下来。

###轻量级指针
####LightRefBase实现原理
```CPP
173template <class T>
174class LightRefBase
175{
176public:
177    inline LightRefBase() : mCount(0) { }
178    inline void incStrong(__attribute__((unused)) const void* id) const {
179        android_atomic_inc(&mCount);
180    }
181    inline void decStrong(__attribute__((unused)) const void* id) const {
182        if (android_atomic_dec(&mCount) == 1) {
183            delete static_cast<const T*>(this);
184        }
185    }
186    //! DEBUGGING ONLY: Get current strong ref count.
187    inline int32_t getStrongCount() const {
188        return mCount;
189    }
190
191    typedef LightRefBase<T> basetype;
192
193protected:
194    inline ~LightRefBase() { }
195
196private:
197    friend class ReferenceMover;
198    inline static void renameRefs(size_t n, const ReferenceRenamer& renamer) { }
199    inline static void renameRefId(T* ref,
200            const void* old_id, const void* new_id) { }
201
202private:
203    mutable volatile int32_t mCount;
204};
```
模板类T必须继承LIghtRefBase类。
####sp中关于轻量级指针的实现
```CPP
template <typename T>
class sp
{
public:
    typedef typename RefBase::weakref_type weakref_type;
    
    inline sp() : m_ptr(0) { }

    sp(T* other);
    sp(const sp<T>& other);
    template<typename U> sp(U* other);
    template<typename U> sp(const sp<U>& other);

    ~sp();
    
    // Assignment

    sp& operator = (T* other);
    sp& operator = (const sp<T>& other);
    
    template<typename U> sp& operator = (const sp<U>& other);
    template<typename U> sp& operator = (U* other);
    
    //! Special optimization for use by ProcessState (and nobody else).
    void force_set(T* other);
    
    // Reset
    
    void clear();
    
    // Accessors

    inline  T&      operator* () const  { return *m_ptr; }
    inline  T*      operator-> () const { return m_ptr;  }
    inline  T*      get() const         { return m_ptr; }

    // Operators
        
    COMPARE(==)
    COMPARE(!=)
    COMPARE(>)
    COMPARE(<)
    COMPARE(<=)
    COMPARE(>=)

private:    
    template<typename Y> friend class sp;
    template<typename Y> friend class wp;

    // Optimization for wp::promote().
    sp(T* p, weakref_type* refs);
    
    T*              m_ptr;
};
```
RefBase::weakref_type类用来维护它所引用的对象的强引用技术和弱引用计数。
sp类中关于轻量级指针的构造函数：
```CPP
template<typename T>
sp<T>::sp(T* other)
    : m_ptr(other)
{
    if (other) other->incStrong(this);
}

template<typename T>
sp<T>::sp(const sp<T>& other)
    : m_ptr(other.m_ptr)
{
    if (m_ptr) m_ptr->incStrong(this);
}
```
incStrong函数是调用的LightRefBase中的相应函数，这是由于类型T是继承了LightRefBase.

