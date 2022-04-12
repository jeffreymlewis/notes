## Terraform Notes
Here are some random notes regarding terraform.

### Find unused variables
This simple command emits the names of terraform variables which are not referenced in `main.tf`.

```shell
for i in $(grep -E '^variable' variables.tf | awk -F\" '{print $2}'); do
  grep $i main.tf > /dev/null || echo $i
done
```
