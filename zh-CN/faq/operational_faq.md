---
id: operational_faq
title: Operational FAQ
sidebar_label: Operational FAQ
---

# 操作 FAQ

### 为什么我启用多进程程序失败了？

Milvus 在运行过程中，能够实现多进程操作，但在实现时需满足一定条件：

- 程序执行时主进程中没有创建 client
- 每个子进程分别创建 client 进行操作

以下为正确实现多进程的示例。当表名为 TABLE_NAME，且已插入 vector_1 的表存在时，直接在主程序中直接调用该函数，两个 insert 进程和一个 search 进程同时执行，且能获得正确结果。其中需注意的是，search 的结果与当前正在 insert 的向量无关。

```shell
def test_add_vector_search_multiprocessing():
    '''
	target: test add vectors, and search it with multiprocessing
	method: set vectors_1[0] as query vectors
	expected: status ok and result length is 1
 	'''
    nq = 1000
    vectors_1 = gen_vec_list(nq)
    vectors_2 = gen_vec_list(nq)

 	def add_vectors_search(idx):
        if idx == 0:
            MILVUS = Milvus()
            connect_server(MILVUS)
            status = add_vec_to_milvus(MILVUS, vectors_1)
            print("add", i, "finished")
            assert status.OK()
        elif idx == 1:
            MILVUS = Milvus()
            connect_server(MILVUS)
            status, result = MILVUS.search_vectors(TABLE_NAME, 1, NPROBE, [vectors_1[0]])
            print(result)
            assert status.OK()
            assert len(result) == 1
  		else:
            MILVUS = Milvus()
            connect_server(MILVUS)
            status = add_vec_to_milvus(MILVUS, vectors_2)
            print("add", i, "finished")
            assert status.OK()

    process_num = 3
 	processes = []
    for i in range(process_num):
        p = mp.Process(target=add_vectors_search, args=(i,))
        processes.append(p)
        p.start()
        print("process", i)
    for p in processes:
        p.join()
```

而若主进程中已存在 client（如利用 client 进行建表及插入），再进行多进程的操作，则会造成 client 假死机，最终导致超时。产生该结果的错误程序示例如下所示。

其中 `connect` 即为主进程所起 client，程序将会持续执行，直至 timeout。

```shell
def test_add_vector_search_multiprocessing(self, connect, table):
    '''
	target: test add vectors, and search it with multiprocessing
	method: set vectors_1[0] as query vectors
	expected: status ok and result length is 1
	'''
    nq = 5
    vectors_1 = gen_vectors(nq, dim)
    vectors_2 = gen_vectors(nq, dim)

    status, ids = connect.add_vectors(table, vectors_1)
    time.sleep(3)

    status, count = connect.get_table_row_count(table)
    assert count == 5

  	def add_vectors_search(connect, idx):
        if (idx % 2) == 0:
            status, ids = connect.add_vectors(table, vectors_2)
            assert status.OK()
        else:
            status, result = connect.search_vectors(table, 1, [vectors_1[0]])
            assert status.OK()
            assert len(result) == 1

    process_num = 3
    processes = []
    for i in range(process_num):
        p = Process(target=add_vectors_search, args=(connect, i))
        processes.append(p)
        p.start()
    for p in processes:
        p.join()
```

### 为什么搜索 top K 的向量，结果不到 K 条向量？

在 Milvus 支持的索引类型中，`IVFLAT` 和 `IVF_SQ8` 是基于 k-means 空间划分的分桶搜索算法。空间被分为 `nlist` 个桶，导入的向量被分配存储在基于 `nlist` 划分的文件结构中。搜索发生时，只搜索最近似的 `nprobe` 个文件。

如果 `nlist` 和 K 比较大，而 `nprobe` 又足够小时，有可能出现 `nprobe` 文件中的所有向量总数小于 K。当你搜索 top K 向量时，就会出现搜索结果小于 K 条向量的情况。

想要避免这种情况，您可以尝试将 `nprobe` 设置为更大值，或是把 `nlist` 和 K 设置小一点。

### 相关阅读

[产品 FAQ](product_faq.md)