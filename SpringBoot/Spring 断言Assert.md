## Spring 断言Assert

### 一. 介绍

在程序开发的过程中，对传入参数进行合法性检查是非常普遍的，如果检查不符合要求，则通过抛出异常的方式拒绝后续处理。Spring 借鉴 Junit 中的断言概念，提供的 **org.springframework.util.Assert** 类拥有众多按规则对方法入参进行断言的方法，可以满足大部分方法入参检测的要求。这些断言方法在入参不满足要求时就会抛出 **IllegalArgumentException**。

### 二. 源码

```java
public abstract class Assert {
    public static void state(boolean expression, String message) {
        if (!expression) {
          	throw new IllegalStateException(message);
        }
    }
  	
  	public static void state(boolean expression, Supplier<String> messageSupplier) {
        if (!expression) {
          	throw new IllegalStateException(nullSafeGet(messageSupplier));
        }
    }
  
    public static void isTrue(boolean expression, String message) {
        if (!expression) {
          	throw new IllegalArgumentException(message);
        }
    }
  
    public static void isTrue(boolean expression, Supplier<String> messageSupplier) {
        if (!expression) {
          	throw new IllegalArgumentException(nullSafeGet(messageSupplier));
        }
    }
    
    public static void isNull(@Nullable Object object, String message) {
        if (object != null) {
          	throw new IllegalArgumentException(message);
        }
    }
  
    public static void isNull(@Nullable Object object, Supplier<String> messageSupplier) {
        if (object != null) {
          	throw new IllegalArgumentException(nullSafeGet(messageSupplier));
        }
    }
    
    public static void notNull(@Nullable Object object, String message) {
        if (object == null) {
          	throw new IllegalArgumentException(message);
        }
    }
  
    public static void notNull(@Nullable Object object, Supplier<String> messageSupplier) {
        if (object == null) {
          	throw new IllegalArgumentException(nullSafeGet(messageSupplier));
        }
    }
  
    public static void hasLength(@Nullable String text, String message) {
        if (!StringUtils.hasLength(text)) {
          	throw new IllegalArgumentException(message);
        }
    }
  
    public static void hasLength(@Nullable String text, Supplier<String> messageSupplier) {
        if (!StringUtils.hasLength(text)) {
          	throw new IllegalArgumentException(nullSafeGet(messageSupplier));
        }
    }
  
    public static void hasText(@Nullable String text, String message) {
        if (!StringUtils.hasText(text)) {
          	throw new IllegalArgumentException(message);
        }
    }
  
  	public static void hasText(@Nullable String text, Supplier<String> messageSupplier) {
        if (!StringUtils.hasText(text)) {
          	throw new IllegalArgumentException(nullSafeGet(messageSupplier));
        }
    }
  
  	public static void doesNotContain(@Nullable String textToSearch, String substring, String message) {
        if (StringUtils.hasLength(textToSearch) && StringUtils.hasLength(substring) &&
            	textToSearch.contains(substring)) {
          	throw new IllegalArgumentException(message);
        }
    }
	
  	public static void doesNotContain(@Nullable String textToSearch, String substring, Supplier<String> messageSupplier) {
      if (StringUtils.hasLength(textToSearch) && StringUtils.hasLength(substring) &&
          	textToSearch.contains(substring)) {
        	throw new IllegalArgumentException(nullSafeGet(messageSupplier));
      }
		}
  
  	public static void notEmpty(@Nullable Object[] array, String message) {
        if (ObjectUtils.isEmpty(array)) {
          	throw new IllegalArgumentException(message);
        }
    }
  	
    public static void notEmpty(@Nullable Object[] array, Supplier<String> messageSupplier) {
        if (ObjectUtils.isEmpty(array)) {
          	throw new IllegalArgumentException(nullSafeGet(messageSupplier));
        }
    }
  
    public static void noNullElements(@Nullable Object[] array, String message) {
        if (array != null) {
            for (Object element : array) {
                if (element == null) {
                  	throw new IllegalArgumentException(message);
                }
            }
        }
    }
  
    public static void noNullElements(@Nullable Object[] array, Supplier<String> messageSupplier) {
        if (array != null) {
            for (Object element : array) {
                if (element == null) {
                  	throw new IllegalArgumentException(nullSafeGet(messageSupplier));
                }
            }
        }
    }
  	
    public static void notEmpty(@Nullable Collection<?> collection, String message) {
        if (CollectionUtils.isEmpty(collection)) {
          	throw new IllegalArgumentException(message);
        }
    }
    
    public static void notEmpty(@Nullable Collection<?> collection, Supplier<String> messageSupplier) {
        if (CollectionUtils.isEmpty(collection)) {
          	throw new IllegalArgumentException(nullSafeGet(messageSupplier));
        }
    }
  
    public static void noNullElements(@Nullable Collection<?> collection, String message) {
        if (collection != null) {
            for (Object element : collection) {
                if (element == null) {
                  	throw new IllegalArgumentException(message);
                }
            }
        }
    }
  
      public static void noNullElements(@Nullable Collection<?> collection, Supplier<String> messageSupplier) {
        if (collection != null) {
            for (Object element : collection) {
                if (element == null) {
                  	throw new IllegalArgumentException(nullSafeGet(messageSupplier));
                }
            }
        }
    }
  
    public static void notEmpty(@Nullable Map<?, ?> map, String message) {
        if (CollectionUtils.isEmpty(map)) {
          	throw new IllegalArgumentException(message);
        }
    }
  
    public static void notEmpty(@Nullable Map<?, ?> map, Supplier<String> messageSupplier) {
        if (CollectionUtils.isEmpty(map)) {
          	throw new IllegalArgumentException(nullSafeGet(messageSupplier));
        }
    }
  
    public static void isInstanceOf(Class<?> type, @Nullable Object obj, String message) {
				notNull(type, "Type to check against must not be null");
        if (!type.isInstance(obj)) {
          	instanceCheckFailed(type, obj, message);
        }
    }
  
    public static void isInstanceOf(Class<?> type, @Nullable Object obj, Supplier<String> messageSupplier){
        notNull(type, "Type to check against must not be null");
        if (!type.isInstance(obj)) {
          	instanceCheckFailed(type, obj, nullSafeGet(messageSupplier));
        }
    }
    。。。
    
    private static void instanceCheckFailed(Class<?> type, @Nullable Object obj, @Nullable String msg) {
        String className = (obj != null ? obj.getClass().getName() : "null");
        String result = "";
        boolean defaultMessage = true;
        if (StringUtils.hasLength(msg)) {
            if (endsWithSeparator(msg)) {
              	result = msg + " ";
            }
            else {
                result = messageWithTypeName(msg, className);
                defaultMessage = false;
            }
        }
        if (defaultMessage) {
          	result = result + ("Object of class [" + className + "] must be an instance of " + type);
        }
        throw new IllegalArgumentException(result);
    }

      private static void assignableCheckFailed(Class<?> superType, @Nullable Class<?> subType, @Nullable String msg) {
        String result = "";
        boolean defaultMessage = true;
        if (StringUtils.hasLength(msg)) {
          if (endsWithSeparator(msg)) {
            result = msg + " ";
          }
          else {
            result = messageWithTypeName(msg, subType);
            defaultMessage = false;
          }
        }
        if (defaultMessage) {
          result = result + (subType + " is not assignable to " + superType);
        }
        throw new IllegalArgumentException(result);
      }

      private static boolean endsWithSeparator(String msg) {
        return (msg.endsWith(":") || msg.endsWith(";") || msg.endsWith(",") || msg.endsWith("."));
      }

      private static String messageWithTypeName(String msg, @Nullable Object typeName) {
        return msg + (msg.endsWith(" ") ? "" : ": ") + typeName;
      }

      @Nullable
      private static String nullSafeGet(@Nullable Supplier<String> messageSupplier) {
        return (messageSupplier != null ? messageSupplier.get() : null);
      }
}
```

### 三. 示例

```java
public static Sort by(Direction direction, String... properties) {
		Assert.notNull(direction, "Direction must not be null!");
		Assert.notNull(properties, "Properties must not be null!");
		Assert.isTrue(properties.length > 0, "At least one property must be given!");

		return Sort.by(Arrays.stream(properties)//
				.map(it -> new Order(direction, it))//
				.collect(Collectors.toList()));
}
```

