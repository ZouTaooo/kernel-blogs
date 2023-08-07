
<!-- alloc_page  get_zeroed_page     __get_free_page__get_dma_pages
    \            /                   \                 /
     \          /                      get_free_pages
      \        /                     /
       \      /                     /
      alloc_pages    <-------------
          |
          |
          v
   alloc_pages_node -->

```mermaid

graph TB

alloc_page --> alloc_pages
get_zeroed_page --> alloc_pages
__get_free_page -->get_free_pages
__get_dma_pages -->get_free_pages
get_free_pages --> alloc_pages
alloc_pages --> alloc_pages_node
```

