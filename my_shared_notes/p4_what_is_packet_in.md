p4_what_is_packet_in.md

`packet_in`  是 extern 的，也就是希望所有的 硬件 实现 都应该提供给 P4 这个对象，(和`packet_out` 一样)

而且遵循如下格式：

```C
packet_in {
	unsigned nextBitIndex;
	byte[] data;
	unsigned lengthInBits;
	void initialize(byte[] data) {
		this.data = data;
		this.nextBitIndex = 0;
		this.lengthInBits = data.sizeInBytes * 8;
	}
	bit<32> length() { return this.lengthInBits / 8; }
}
```

其支持的基本行为应该有：

```c

extern packet_in {
	void extract<T>(out T headerLvalue);
	void extract<T>(out T variableSizeHeader, in bit<32> varFieldSizeBits);
	T lookahead<T>();
	bit<32> length(); // This method may be unavailable in some architectures
	void advance(bit<32> bits);
}
```