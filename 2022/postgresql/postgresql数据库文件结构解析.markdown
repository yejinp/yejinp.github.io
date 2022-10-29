
|         |      |
|---------|------|
|         |      |
|         |      |
|         |      |

total cost = start-up + run

start-up: the cost expended before the first tuple is fetched.
run: the cost to fetch all tuples.

每个page的大小默认为8kb, 在configure源代码的时候可以通过参数block_size指定；