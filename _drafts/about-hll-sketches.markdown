`APPROX_COUNT_DISTINCT_DS_HLL` can take as input either a regular column (string or number), or a sketch.
If the input is a string (or a number), it assumes that the value of that string is the distinct thing that you want to count. Even if the string is the base64 representation of a sketch, `APPROX_COUNT_DISTINCT_DS_HLL` does not know that.
If the input is a sketch (a COMPLEX type), it applies the sketch merge and estimate directly.
`COMPLEX_DECODE_BASE64` basically restores the magic, it casts the string into a COMPLEX sketch type so that the other version of `APPROX_COUNT_DISTINCT_DS_HLL`  is called.
