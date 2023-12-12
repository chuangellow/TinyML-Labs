# Note on Lab 0

## Paged-lock Memory

在創建 data loader 時，這裡下了一個 ```pin_memory=True``` 的 arguments：

```
dataflow = {}
for split in ['train', 'test']:
  dataflow[split] = DataLoader(
    dataset[split],
    batch_size=512,
    shuffle=(split == 'train'),
    num_workers=0,
    pin_memory=True,
  )
```

回去看 data loader 的定義，可以看到 pin_memory 參數的解釋為：

> pin_memory (bool, optional): If ``True``, the data loader will copy Tensors into device/CUDA pinned memory before returning them.  If your data elements are a custom type, or your :attr:`collate_fn` returns a batch that is a custom type, see the example below.

原因在於我們的資料集是在 CPU 上讀取的，但我們希望在 GPU 上訓練時，我們可以透過設定 ```pin_memory``` 為 ```True``` 來讓 DataLoader 將 sample 讀到 GPU 的 paged-lock memory 上，來加速資料的傳輸。

那首先在講到 paged-lock memory 為何可以加速資料傳輸，就必須先講到 paging 機制，

paging 常見用於作業系統中 memory management 中，透過 paging 機制，可以讓每個 process 有自己的 logical memory，每個 process 會認為自己存取的是一塊連續的記憶體空間，但實際上，他的 logical memory 會分成不同的 page，OS 會維護一個 page table，來查詢說這些 logical memory 切割成的 page 實際對應到記憶體的哪個空間，就能做到不連續的記憶體空間分配。

而當記憶體空間不足時，會將某些 process 閒置的 page 移到 disk 儲存，此步驟稱為 page-out，直到 page 有被需求時，才 page-in 回到記憶體。

因此，在記憶體中，可以被 page-in 和 page-out 的空間，稱為 pageable memory。

反之，被禁止做 page-in 和 page-out 的空間，就稱為 paged-lock memory。

因為 paged-lock memory 空間的資料不會有 disk 來回的存取，latency 就較低。

因此，在這裡，我們可以透過指定資料必須存放在 paged-lock memory，來節省說可能在 host 這端必須將資料從 disk 搬回 memory 的時間。

# Reference

- [Page-Locked Host Memory for Data Transfer](https://leimao.github.io/blog/Page-Locked-Host-Memory-Data-Transfer/)
