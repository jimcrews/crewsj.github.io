### How man roles are there in all namespaces ?

use `--no-headers`, `wc` and `-l` for 'lines'
``` shell
k get roles -A --no-headers | wc -l
```