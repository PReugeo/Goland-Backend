## Helm install

- chart 的仓库（如上所述）
- 本地 chart 压缩包（`helm install foo foo-0.1.1.tgz`）
- 解压后的 chart 目录（`helm install foo path/to/foo`）
- 完整的 URL（`helm install foo https://example.com/charts/foo-1.2.3.tgz`）

## Helm 打包

`Helm package path/to/foo`

## Helm 更新

`helm upgrade {xxx} -n xxx`