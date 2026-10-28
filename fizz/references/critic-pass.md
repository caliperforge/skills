# Critic Pass: Share-Converted Comparisons

The critic flags every function in `Properties.sol` and `handlers/*.sol` that compares a value from `previewRedeem`, `convertToAssets`, `previewWithdraw` or `convertToShares` with an exact assertion and no tolerance, breaking `property-generation.md` section 6. Such a check fails on rounding dust alone, on buggy and fixed code alike, so it catches nothing.

Run from `{PROJECT_ROOT}`. It prints `file:line:function` per hit and nothing on a clean suite:

```sh
awk '
function flush() {
  if (name != "" && !marked &&
      body ~ /(previewRedeem|convertToAssets|previewWithdraw|convertToShares)\(/ &&
      body ~ /[^A-Za-z0-9_](eq|lte|gte|lt|gt)\(|[<>]=/ &&
      body !~ /[Tt]olerance|TOLERANCE/)
    print file ":" line ":" name
  name = ""; body = ""
}
FNR == 1 { flush(); file = FILENAME; doc = "" }
/^[ \t]*(\/\/|\/\*|\*)/ { doc = doc $0; next }
{
  code = $0; gsub(/"[^"]*"/, "", code); sub(/\/\/.*/, "", code)
  if (code ~ /(^|[^A-Za-z0-9_])function[ \t]/) {
    flush()
    match(code, /function[ \t]+[A-Za-z0-9_]+/)
    name = substr(code, RSTART, RLENGTH); sub(/function[ \t]+/, "", name)
    line = FNR; marked = (doc ~ /rounding-direction/)
  }
  body = body " " code
  doc = ""
}
END { flush() }
' {SUITE_DIR}/Properties.sol {SUITE_DIR}/handlers/*.sol
```

Clear a hit in one of three ways:

- Measure the moved value with the asset's `balanceOf` deltas through `Snapshots.sol`.
- Compare against a stated tolerance whose name contains `tolerance`: a few wei per conversion times the conversions in the call, with the count in the natspec.
- If the inequality is the ERC-4626 rounding direction (roundtrip-rounding-analyst Pattern D or E), add `/// @dev rounding-direction` directly above the function.
