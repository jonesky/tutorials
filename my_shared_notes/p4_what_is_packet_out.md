p4_what_is_packet_out.md

```C
packet_out {
	byte[] data;
	unsigned lengthInBits;
	void initializeForWriting() {
		this.data.clear();
		this.lengthInBits = 0;
	}
/// Append data to the packet. Type T must be a header, header
/// stack, header union, or struct formed recursively from those types

	void emit<T>(T data) {
		if (isHeader(T))
			if(data.valid$) {
				this.data.append(data);
				this.lengthInBits += data.lengthInBits;
		}
		else if (isHeaderStack(T))
			for (e : data)
				emit(e);
			else if (isHeaderUnion(T) || isStruct(T))
				for (f : data.fields$)
					emit(e.f)
// Other cases for T are illegal
}
```