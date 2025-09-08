# B+树实现详解

## 概述

这是一个完整的B+树数据结构实现，支持插入、查找、删除和范围查询等操作,也是DRAM下PLIN-index的溢出树结构，框架参考了Rucbase的index模块。B+树是一种自平衡的树数据结构，广泛应用于数据库系统的索引实现。

## 核心配置参数

```cpp
constexpr static uint32_t BTREE_PLUS_PAGESIZE = 256;  // 页面大小为256字节
const int cardinality = (BTREE_PLUS_PAGESIZE-sizeof(header))/sizeof(Record);  // 每页最多14条记录
```

## 数据结构定义

### 1. Record 结构体
```cpp
struct Record{
    _key_t key;      // 键值
    char *val;       // 指向数据的指针
    Record(_key_t k = INT64_MAX, char *v = NULL): key(k), val(v){};
    bool operator< (const Record& r) const {return key < r.key;}
};
```

**说明：** Record是B+树中存储的基本数据单元，包含键值对。在叶子节点中，val指向实际数据；在内部节点中，val指向子节点。

### 2. Header 类
```cpp
class header{
private:
    bool is_leaf;        // 是否为叶子节点
    bool is_root;        // 是否为根节点
    page *next_leaf;     // 指向下一个叶子节点（用于范围查询）
    page *prev_leaf;     // 指向前一个叶子节点
    page *parent;        // 指向父节点
    uint32_t level;      // 节点层级
    int16_t record_num;  // 当前记录数量
    int8_t is_deleted;   // 删除标记
    uint64_t unused;     // 保留字段
    
public:
    header(){
        is_leaf = false;
        next_leaf = nullptr;
        prev_leaf = nullptr;
        parent = nullptr;
        level = 0;
        record_num = 0;
        is_deleted = 0;
        unused = 0;
    }
};
```

**说明：** Header存储节点的元数据信息，包括节点类型、父子关系、记录数量等。

### 3. Page 类

Page类是B+树的核心，代表树中的一个节点（页面）。

```cpp
class page{
private:
    header hdr;                    // 页面头部信息
    Record records[cardinality];   // 记录数组，最多存储14条记录
    int IX_NO_PAGE = -1;          // 无效页面标识
    
public:
    // 构造函数：创建指定层级的页面
    page(uint32_t level){
        hdr.level = level;
        // 初始化所有记录为空
        for(int i = 0; i < cardinality; i++){
            records[i].key = FREE_FLAG;
            records[i].val = nullptr;
        }
    }
```

#### 核心查找方法

```cpp
// 查找第一个大于等于target_key的位置
int lower_bound(_key_t target_key) {
    int i;
    for(i = 0; i < hdr.record_num; i++){
        if(records[i].key >= target_key)
            break;
    }
    return i;
}

// 查找第一个大于target_key的位置
int upper_bound(_key_t target_key){
    int i;
    for(i = 1; i < hdr.record_num; i++){
        if(records[i].key > target_key)
            break;
    }
    return i;
}
```

#### 叶子节点查找

```cpp
// 在叶子节点中查找指定键值
bool leaf_lookup(_key_t target_key, _payload_t &payload){
    int id = lower_bound(target_key);
    int size = count();
    if(id != size && records[id].key == target_key){
        payload = reinterpret_cast<_payload_t>(records[id].val);    
        return true;
    }
    return false;
}
```

#### 内部节点查找

```cpp
// 在内部节点中根据key查找对应的子节点
page *inner_lookup(_key_t target_key){
    int id = upper_bound(target_key) - 1;
    return reinterpret_cast<page*>(records[id].val);
}
```

#### 插入操作

```cpp
// 在节点中插入键值对（支持更新）
int insert_in_node(_key_t key, _payload_t &payload){
    int id = lower_bound(key);
    
    // 如果键已存在，则更新值
    if(records[id].key == key && records[id].val != nullptr) {
        records[id].val = reinterpret_cast<char*>(payload);
        payload = INT64_MAX;  // 标记为更新操作
        return hdr.record_num;
    }
    
    // 插入新记录
    if(id < 0 || id > hdr.record_num) return hdr.record_num;
    
    assert(hdr.record_num < cardinality);
    
    // 移动后续元素为新记录腾出空间
    for(int pos = hdr.record_num - 1; pos >= id; pos--){
        records[pos + 1].key = records[pos].key;
        records[pos + 1].val = records[pos].val;
    }
    
    records[id].key = key;
    records[id].val = reinterpret_cast<char*>(payload);
    hdr.record_num++;
    return hdr.record_num;
}
```

#### 节点分裂

```cpp
// 当节点满时进行分裂操作
page* split(page *node){
    page* sibling = new page(node->hdr.level);
    sibling->hdr.is_leaf = node->hdr.is_leaf;
    sibling->hdr.parent = nullptr;
    sibling->hdr.record_num = 0;
    
    // 如果是叶子节点，需要维护叶子节点链表
    if(sibling->hdr.is_leaf){
        page *next_node = node->hdr.next_leaf;
        if(next_node == nullptr){
            node->hdr.next_leaf = sibling;
            sibling->hdr.prev_leaf = node;
            sibling->hdr.next_leaf = nullptr;
        }else{
            sibling->hdr.prev_leaf = node;
            node->hdr.next_leaf = sibling;
            next_node->hdr.prev_leaf = sibling;
            sibling->hdr.next_leaf = next_node;
        }
    }

    // 分割记录：左节点保留前半部分，右节点获得后半部分
    int right = node->hdr.record_num / 2;     // 右节点记录数
    int left = node->hdr.record_num - right;  // 左节点记录数
    
    // 将后半部分记录移动到新节点
    for(int k = 0; k < right; k++){
        sibling->records[k].key = node->records[k + left].key;
        sibling->records[k].val = node->records[k + left].val;
        node->records[k + left].val = nullptr;
        sibling->hdr.record_num++;
        node->hdr.record_num--;
    }

    // 如果不是叶子节点，需要维护父子关系
    if(!node->hdr.is_leaf) {
        for (int i = 0; i < sibling->hdr.record_num; i++) {
            maintain_child(sibling, i);
        }
    }
    return sibling;
}
```

## B+树主类实现

### 构造函数

```cpp
btree_plus::btree_plus(void** addr, bool init) : addr(addr) {
    if (init) {
        // 初始化新的B+树
        root = new page(0);
        root->hdr.is_leaf = true;
        *addr = root;
    }
    else {
        // 从已存在的地址恢复B+树
        root = reinterpret_cast<page*>(*addr);
    }
}
```

### 查找操作

```cpp
// 查找叶子页面
page* btree_plus::find_leaf_page(_key_t key){
    page *ret = root;
    // 从根节点开始，沿着树向下查找，直到到达叶子节点
    while(!ret->hdr.is_leaf){
        ret = ret->inner_lookup(key);
    }
    return ret;
}

// 查找指定键值
bool btree_plus::find(_key_t key, _payload_t &payload){
    page *leaf = find_leaf_page(key);
    return leaf->leaf_lookup(key, payload);
}
```

### 插入操作

```cpp
// 插入或更新键值对
int btree_plus::upsert(_key_t key, _payload_t payload){
    page *leaf = find_leaf_page(key);
    int num = leaf->hdr.record_num;
    
    // 尝试在叶子节点中插入
    if(leaf->insert_in_node(key, payload) == num){
        if(payload == INT64_MAX) return 3;  // 更新操作
        else return 0;  // 插入失败
    };

    // 如果叶子节点满了，需要分裂
    if(leaf->hdr.record_num >= cardinality){
        page* sibling = leaf->split(leaf);
        insert_into_parent(leaf, sibling->records[0].key, sibling);
    }
    return 4;  // 插入成功
}
```

### 向父节点插入

```cpp
// 当子节点分裂后，需要在父节点中插入新的键值对
void btree_plus::insert_into_parent(page *old_node, _key_t key, page *new_node){
    // 如果分裂的是根节点，创建新的根节点
    if(old_node == root){
        page *new_root = new page(0);
        new_root->hdr.is_leaf = false;
        new_root->hdr.parent = nullptr;
        new_root->hdr.record_num = 0;

        // 设置新根节点的两个子节点
        new_root->records[0].key = old_node->records[0].key;
        new_root->records[0].val = reinterpret_cast<char*>(old_node);
        new_root->records[1].key = key;
        new_root->records[1].val = reinterpret_cast<char*>(new_node);
        
        old_node->hdr.parent = new_root;
        new_node->hdr.parent = new_root;
        new_root->hdr.record_num = 2;
        set_new_root(new_root);
        return;
    }

    // 在父节点中插入新的键值对
    page *father = old_node->hdr.parent;
    int child_idx = father->find_child_rank(old_node);
    new_node->hdr.parent = father;
    father->insert_pair(child_idx + 1, key, reinterpret_cast<char*>(new_node));
    
    // 如果父节点也满了，需要递归分裂
    if(father->hdr.record_num >= cardinality){
       page* father_sibling = father->split(father);
       insert_into_parent(father, father_sibling->records[0].key, father_sibling);
    }else{
        new_node->hdr.parent = father;  
    }
}
```

### 删除操作

```cpp
// 删除指定键值
bool btree_plus::delete_entry(_key_t key){
    page *leaf = find_leaf_page(key);
    int old_num = leaf->hdr.record_num;
    int new_num = leaf->remove(key);
    
    maintain_parent(leaf);  // 维护父节点信息
    
    if(old_num != new_num) {
        coalesce_or_redistribute(leaf);  // 处理节点下溢
        return true;
    }else{
        return false;
    }
}
```

### 合并或重分布

```cpp
// 处理节点删除后可能的下溢情况
bool btree_plus::coalesce_or_redistribute(page *node){
    // 如果是根节点，调整根节点
    if(node == root){
        return adjust_root(node);
    }
    // 如果节点记录数满足最小要求，无需处理
    else if(node->hdr.record_num >= (cardinality + 1)/2 ){
        return false;
    }

    // 查找兄弟节点
    page* father = node->hdr.parent;
    page *neighbor;
    bool is_left_neighbor = true;
    int index = father->find_child_rank(node);
    
    if(index == 0){
        neighbor = reinterpret_cast<page *>(father->records[1].val);
        is_left_neighbor = false;
    }else{
        neighbor = reinterpret_cast<page *>(father->records[index-1].val);
    }

    // 判断是重分布还是合并
    if(node->hdr.record_num + neighbor->hdr.record_num >= (cardinality + 1)){
        redistribute(neighbor, node, father, index);  // 重分布
        return false;
    }else{
        coalesce(&neighbor, &node, &father, index, is_left_neighbor);  // 合并
        return true;
    }
}
```

### 范围查询

```cpp
// 范围查询：查找指定范围内的所有键值对
void btree_plus::range_query(_key_t lower_bound, _key_t upper_bound, 
                           std::vector<std::pair<_key_t, _payload_t>>& answers){
    page* curr = find_leaf_page(lower_bound);  // 找到起始叶子节点
    
    // 沿着叶子节点链表遍历
    while(curr != nullptr){
        for(int i = 0; i < curr->hdr.record_num; i++){
            if(curr->records[i].key >= lower_bound && 
               curr->records[i].key <= upper_bound && 
               curr->records[i].val != nullptr){
                answers.push_back(std::make_pair(curr->records[i].key, 
                                               reinterpret_cast<_payload_t>(curr->records[i].val)));
            }
        }
        curr = curr->hdr.next_leaf;  // 移动到下一个叶子节点
    }
}
```

## 算法复杂度分析

### 时间复杂度
- **查找**: $O(\log_m n)$，其中 $m$ 是节点的最大子节点数，$n$ 是总记录数
- **插入**: $O(\log_m n)$，最坏情况下需要分裂路径上的所有节点
- **删除**: $O(\log_m n)$，最坏情况下需要合并路径上的节点
- **范围查询**: $O(\log_m n + k)$，其中 $k$ 是结果集大小

### 空间复杂度
- 每个节点最多存储 14 条记录（基于256字节页面大小）
- 树的高度为 $O(\log_m n)$
- 总空间复杂度：$O(n)$

## 特性总结

1. **自平衡**: 通过分裂和合并操作维护树的平衡
2. **高效范围查询**: 叶子节点通过链表连接，支持高效的范围扫描
3. **页面式存储**: 固定大小的页面设计，适合磁盘存储
4. **支持并发**: 结构设计便于实现锁机制
5. **缓存友好**: 连续的内存访问模式

这个B+树实现是一个功能完整的数据结构，适用于数据库索引、文件系统等需要高效键值查找和范围查询的场景。