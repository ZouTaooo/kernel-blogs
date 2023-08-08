```mermaid

graph TB

alloc_page ---> alloc_pages
get_zeroed_page ---> alloc_pages
__get_free_page -->get_free_pages
__get_dma_pages -->get_free_pages
get_free_pages --> alloc_pages
alloc_pages --> alloc_pages_node

```

--------------------------------------

```mermaid

graph LR

A[order==0 ? ] -->|yes| B[hot-cold-list]
A -->|no| C[__rmqueue]
C --> D[prep_new_page]
B --> D

```


-----------------------------------

```mermaid
graph

A[__rmqueue]
B[1.__rmqueue_smallest]
C[expand]
D[__rmqueue_fallback]
E[rmv_page_order]
A -->| | B
A -->|failed| D
B -->|1| E
B -->|2| C

```

-------------------------------------

```mermaid
graph TB

A[__free_one_page]
B[__free_pages_ok]
C[free_hot_page]
D[__free_pages]
E[free_one_page]

free_page --> free_pages
free_pages --> D
__free_page --> D

D -->|order==0| C
D -->|order>0| B
B --> E
E --> A


```

----------------------------------------------

```mermaid
graph 

A[vmalloc]
B[__vmalloc]
C[__vmalloc_node]
D[get_vm_area_node]
E[__vmalloc_area_node]
F[__get_vm_area_node]
G[kmalloc_node]
H[alloc_pages_node]
I[map_vm_area]

A --> B
B --> C
C --> |alloc vm_struct|D --> F --> G
C --> |alloc pages and map|E
E --> H
E --> I

%% E --> |pages >= PAGE_SIZE|C
%% E --> |pages < PAGE_SIZE|G
```
----------------------------

:::mermaid

graph

A[vfree]
B[__vunmap]
C[remove_vm_area]
D[__free_page]
E[kfree or vfree]
F[__remove_vm_area]
G[unmap_vm_area]
H[unmap_kernel_range]

A --> B --> C --> F --> G --> H
B --> D
B --> E

:::
--------------------------
:::mermaid

flowchart LR

A[直接映射]
B[vmalloc]
C[持久映射]
D[固定映射]

A -.- B -.- C -.- D

:::