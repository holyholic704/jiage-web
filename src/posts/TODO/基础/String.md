

```java
private final char value[];

/** Cache the hash code for the string */
private int hash; // Default to 0

/**
 * Class String is special cased within the Serialization Stream Protocol.
 *
 * A String instance is written into an ObjectOutputStream according to
 * <a href="{@docRoot}/../platform/serialization/spec/output.html">
 * Object Serialization Specification, Section 6.2, "Stream Elements"</a>
 */
private static final ObjectStreamField[] serialPersistentFields =
    new ObjectStreamField[0];
```

```java
public String() {
    this.value = "".value;
}

public String(String original) {
    this.value = original.value;
    this.hash = original.hash;
}

public String(char value[]) {
    this.value = Arrays.copyOf(value, value.length);
}

public String(char value[], int offset, int count) {
    if (offset < 0) {
        throw new StringIndexOutOfBoundsException(offset);
    }
    if (count <= 0) {
        if (count < 0) {
            throw new StringIndexOutOfBoundsException(count);
        }
        if (offset <= value.length) {
            this.value = "".value;
            return;
        }
    }
    // Note: offset or count might be near -1>>>1.
    if (offset > value.length - count) {
        throw new StringIndexOutOfBoundsException(offset + count);
    }
    this.value = Arrays.copyOfRange(value, offset, offset+count);
}

public String(int[] codePoints, int offset, int count) {
    if (offset < 0) {
        throw new StringIndexOutOfBoundsException(offset);
    }
    if (count <= 0) {
        if (count < 0) {
            throw new StringIndexOutOfBoundsException(count);
        }
        if (offset <= codePoints.length) {
            this.value = "".value;
            return;
        }
    }
    // Note: offset or count might be near -1>>>1.
    if (offset > codePoints.length - count) {
        throw new StringIndexOutOfBoundsException(offset + count);
    }

    final int end = offset + count;

    // Pass 1: Compute precise size of char[]
    int n = count;
    for (int i = offset; i < end; i++) {
        int c = codePoints[i];
        if (Character.isBmpCodePoint(c))
            continue;
        else if (Character.isValidCodePoint(c))
            n++;
        else throw new IllegalArgumentException(Integer.toString(c));
    }

    // Pass 2: Allocate and fill in char[]
    final char[] v = new char[n];

    for (int i = offset, j = 0; i < end; i++, j++) {
        int c = codePoints[i];
        if (Character.isBmpCodePoint(c))
            v[j] = (char)c;
        else
            Character.toSurrogates(c, v, j++);
    }

    this.value = v;
}

public String(byte bytes[], int offset, int length, String charsetName)
        throws UnsupportedEncodingException {
    if (charsetName == null)
        throw new NullPointerException("charsetName");
    checkBounds(bytes, offset, length);
    this.value = StringCoding.decode(charsetName, bytes, offset, length);
}

public String(byte bytes[], int offset, int length, Charset charset) {
    if (charset == null)
        throw new NullPointerException("charset");
    checkBounds(bytes, offset, length);
    this.value =  StringCoding.decode(charset, bytes, offset, length);
}

public String(byte bytes[], String charsetName)
        throws UnsupportedEncodingException {
    this(bytes, 0, bytes.length, charsetName);
}

public String(byte bytes[], Charset charset) {
    this(bytes, 0, bytes.length, charset);
}

public String(byte bytes[], int offset, int length) {
    checkBounds(bytes, offset, length);
    this.value = StringCoding.decode(bytes, offset, length);
}

public String(byte bytes[]) {
    this(bytes, 0, bytes.length);
}

public String(StringBuffer buffer) {
    synchronized(buffer) {
        this.value = Arrays.copyOf(buffer.getValue(), buffer.length());
    }
}

public String(StringBuilder builder) {
    this.value = Arrays.copyOf(builder.getValue(), builder.length());
}

String(char[] value, boolean share) {
    // assert share : "unshared not supported";
    this.value = value;
}
```

```java
public class StringTest {

    public static void main(String[] args) {
        String s = "good";
        StringBuilder sb1 = new StringBuilder("good");
        StringBuilder sb2 = new StringBuilder(s);

        System.out.println(s == sb1.toString());
        System.out.println(s == sb2.toString());
        System.out.println(s.equals(sb1.toString()));
        System.out.println(s.equals(sb2.toString()));

        System.out.println(Integer.toBinaryString(-2));
        System.out.println(Integer.toBinaryString(-2 >>> 16));
        System.out.println(Integer.toBinaryString(16));
        System.out.println(Integer.numberOfLeadingZeros(16));
        System.out.println(Integer.numberOfTrailingZeros(16));

        System.out.println(Integer.toBinaryString(65536));
        System.out.println(Integer.toBinaryString(65535));

        System.out.println(Arrays.toString("龥".toCharArray()));

        System.out.println(Integer.toHexString(Character.codePointAt("我", 0)));
        System.out.println(Integer.toBinaryString(Character.codePointAt("我", 0)));
        System.out.println(Integer.toBinaryString(Character.codePointAt("龥", 0)));
        System.out.println(Integer.toBinaryString(0xffffffff));
        System.out.println(Character.codePointAt("龥", 0));

        char c = '\uffff';
        System.out.println(c);
    }
}
```