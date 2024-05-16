# 基础概念

## JavaBean

如果读写方法符合以下这种命名规范：

```java
public class Abc {
  private String xyz;
  // 读方法:
  public Type getXyz()
  // 写方法:
  public void setXyz(Type value)
}
```

那么这种`class`被称为`JavaBean`

