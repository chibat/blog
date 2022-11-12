# Java の lambda 内のチェック例外のウザさをなんとかする(2019-11-29)

Java の Stream API などを使っていると、lambda 内のチェック例外がうざいです。

例えば、次のようなコードは、コンパイルエラーになります。

```java
Stream.of("http://example.com").map(string -> {
  return new URL(string); // MalformedURLException を throw する可能性があるのでコンパイルエラー
});
```

catch して補足することを強要されます。そのため、RuntimeException にラップする static メソッド `wrap` を作成してみました。利用すると次のようになります。このコードは、コンパイルエラーにはなりません。

```java
Stream.of("http://example.com").map(string -> wrap(() -> {
  return new URL(string);
}));
```

この `wrap` メソッドの実装は、次のようになります。

```java
import java.io.IOException;
import java.io.UncheckedIOException;

/**
 * チェック例外を RuntimeException 系の Exception にラップする。
 */
public class CheckedExceptionWrapper {

  /**
   * 戻り値があるパターン
   */
  public static <R> R wrap(final ThrowableSupplier<R> supplier) {
    try {
      return supplier.get();
    } catch (final RuntimeException cause) {
      // 非チェック例外は、そのまま throw
      throw cause;
    } catch (final IOException cause) {
      // IOException は UncheckedIOException で wrap
      throw new UncheckedIOException(cause);
    } catch (final Throwable cause) {
      // RuntimeException を継承した独自の Exception を使った方が良いかも
      throw new RuntimeException(cause);
    }
  }

  /**
   * 戻り値がないパターン
   */
  public static void wrap(final ThrowableRunner runner) {
    try {
      runner.run();
    } catch (final RuntimeException cause) {
      throw cause;
    } catch (final IOException cause) {
      throw new UncheckedIOException(cause);
    } catch (final Throwable cause) {
      throw new RuntimeException(cause);
    }
  }

  @FunctionalInterface
  public interface ThrowableSupplier<R> {
    R get() throws Throwable;
  }

  @FunctionalInterface
  public interface ThrowableRunner {
    void run() throws Throwable;
  }
}
```

RuntimeException にラップするのではなく、RuntimeException を継承した独自の Exception でラップした方が良いかもしれません。
大域で catch している部分で判別しやすいかもしれません。

以上
