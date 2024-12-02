# AppId、AppKey、AppSecret

- AppId：应用唯一标识，即用户在系统中的唯一标识
  - 一般不允许重新生成
- AppKey：公钥，常用来表示用户的权限
  - 一个用户可以有多个 AppKey，但一般只会有一个
  - 允许重新生成
- AppSecret：私钥，常与 AppKey 成对出现
  - AppId 和 AppKey 泄露了一般不会有太大问题，但一定要保证 AppSecret 的数据安全

## 简化

### 省去 AppId 或 AppKey

对于只有一套权限的系统，可以省去 AppId 或 AppKey，只需要其中一个就可以代表用户

### 只使用其中一个

对于不需要权限控制，但需要记录用户的相关调用情况时，可以只使用其中一个参数，一般是 AppSecret
