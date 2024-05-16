## CP 功能

```tsx
# https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands

！！！重要的提示！！！# 要求容器中存在“tar”二进制文件 # 图像。如果 'tar' 不存在，'kubectl cp' 将失败。# # 对于高级用例，例如符号链接、通配符扩展或 # 文件模式保存，请考虑使用“kubectl exec”。
# 将 /tmp/foo 本地文件复制到命名空间中远程 pod 中的 /tmp/bar
tar cf - /tmp/foo | kubectl exec -i -n <some-namespace> <some-pod> -- tar xf - -C /tmp/bar

将 /tmp/foo 从远程 pod 复制到本地的 /tmp/bar
kubectl exec -n <some-namespace> <some-pod> -- tar cf - /tmp/foo | tar xf - -C /tmp/bar

将 /tmp/foo_dir 本地目录复制到默认命名空间中远程 pod 中的 /tmp/bar_dir
kubectl cp /tmp/foo_dir <some-pod>:/tmp/bar_dir

将 /tmp/foo 本地文件复制到特定容器中远程 pod 中的 /tmp/bar
kubectl cp /tmp/foo <some-pod>:/tmp/bar -c <specific-container>

将 /tmp/foo 本地文件复制到命名空间中远程 pod 中的 /tmp/bar
kubectl cp /tmp/foo <some-namespace>/<some-pod>:/tmp/bar

将 /tmp/foo 从远程 pod 复制到本地的 /tmp/bar
kubectl cp <some-namespace>/<some-pod>:/tmp/foo /tmp/bar
```

### python api 实现

```python
from kubernetes import client, config
from kubernetes.stream import stream
import tarfile
from tempfile import TemporaryFile
from kubernetes.client.rest import ApiException
from os import path


def copy_file_inside_pod(api_instance, pod_name, src_path, dest_path, namespace='default'):
    """
    This function copies a file inside the pod
    :param api_instance: coreV1Api()
    :param name: pod name
    :param ns: pod namespace
    :param source_file: Path of the file to be copied into pod
    :return: nothing
    """

    try:
        exec_command = ['tar', 'xvf', '-', '-C', '/']
        api_response = stream(api_instance.connect_get_namespaced_pod_exec, pod_name, namespace,
                              command=exec_command,
                              stderr=True, stdin=True,
                              stdout=True, tty=False,
                              _preload_content=False)

        with TemporaryFile() as tar_buffer:
            with tarfile.open(fileobj=tar_buffer, mode='w') as tar:
                tar.add(src_path, dest_path)

            tar_buffer.seek(0)
            commands = [tar_buffer.read()]
            while api_response.is_open():
                api_response.update(timeout=1)
                if api_response.peek_stdout():
                    print('STDOUT: {0}'.format(api_response.read_stdout()))
                if api_response.peek_stderr():
                    print('STDERR: {0}'.format(api_response.read_stderr()))
                if commands:
                    c = commands.pop(0)
                    api_response.write_stdin(c.decode())
                else:
                    break
            api_response.close()
    except ApiException as e:
        print('Exception when copying file to the pod: {0} \n'.format(e))


def copy_file_from_pod(api_instance, pod_name, src_path, dest_path, namespace="default"):
    """
    :param pod_name:
    :param src_path: /tmp/123.txt
    :param dest_path: /root/
    :param namespace:
    :return:
    """

    dir = path.dirname(src_path)
    bname = path.basename(src_path)
    exec_command = ['/bin/sh', '-c', 'cd {src}; tar cf - {base}'.format(src=dir, base=bname)]

    with TemporaryFile() as tar_buffer:
        resp = stream(api_instance.connect_get_namespaced_pod_exec, pod_name, namespace,
                      command=exec_command,
                      stderr=True, stdin=True,
                      stdout=True, tty=False,
                      _preload_content=False)

        while resp.is_open():
            resp.update(timeout=1)
            if resp.peek_stdout():
                out = resp.read_stdout()
                # print("STDOUT: %s" % len(out))
                tar_buffer.write(out.encode('utf-8'))
            if resp.peek_stderr():
                print('STDERR: {0}'.format(resp.read_stderr()))
        resp.close()

        tar_buffer.flush()
        tar_buffer.seek(0)

        with tarfile.open(fileobj=tar_buffer, mode='r:') as tar:
            subdir_and_files = [
                tarinfo for tarinfo in tar.getmembers()
            ]
            tar.extractall(path=dest_path, members=subdir_and_files)


def main():
    # Configs can be set in Configuration class directly or using helper
    # utility. If no argument provided, the config will be loaded from
    # default location.
    configuration = client.Configuration()
    configuration.host = 'https://10.39.32.x:6443'
    configuration.verify_ssl = False
    configuration.api_key = {"authorization": "Bearer " + 'xxxx'}
    api = client.CoreV1Api(client.ApiClient(configuration))
    pod_name = 'test-lm-web-756cb6c47c-rnbjb'  # Pod name to/from you want to copy file
    src_path = '/tmp/77777.txt'  # File/folder you want to copy
    dest_path = '/tmp/123.txt'  # Destination path on which you want to copy the file/folder

    # copy_file_from_pod(api_instance=api, pod_name=pod_name, src_path=src_path, dest_path=dest_path, namespace='me-test-project-tjx')
    copy_file_inside_pod(api_instance=api, pod_name=pod_name, src_path=src_path, dest_path=dest_path,
                         namespace='me-test-project-tjx')


if __name__ == '__main__':
    main()
```

