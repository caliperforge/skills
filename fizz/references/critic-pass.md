# Critic Pass

## Share-Converted Comparisons

The critic flags every function in `Properties.sol` and `handlers/*.sol` that compares a value from `previewRedeem`, `convertToAssets`, `previewWithdraw` or `convertToShares` with an exact assertion and no tolerance, breaking `property-generation.md` section 6.

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

## Unbounded Stand-ins

The critic flags every `contract Mock*` or `contract StandIn*` with no `// stand-in limits:` block directly above it, breaking `property-generation.md` section 7, and every function in `handlers/*.sol` named `<var>_...` where `Base.sol` declares `<var>` with a flagged type. The scaffolded `utils/MockERC20.sol` is skipped.

Run from `{PROJECT_ROOT}`. It prints `file:line:Contract` per stand-in, then `file:line:function` per handler, and nothing on a clean suite:

```sh
awk '
FNR == 1 { doc = "" }
/^[ \t]*(\/\/|\/\*|\*)/ { doc = doc $0; next }
!pass && /^[ \t]*contract[ \t]+(Mock|StandIn)/ && doc !~ /stand-in limits:/ {
  match($0, /contract[ \t]+[A-Za-z0-9_]+/)
  name = substr($0, RSTART, RLENGTH); sub(/contract[ \t]+/, "", name)
  print FILENAME ":" FNR ":" name; bad[name] = 1
}
pass == 2 && ($1 in bad) && /[;=]/ {
  decl = $0; sub(/[ \t]*[;=].*/, "", decl); n = split(decl, w); vars[w[n]] = 1
}
pass == 3 && match($0, /function[ \t]+[A-Za-z0-9_]+/) {
  name = substr($0, RSTART, RLENGTH); sub(/function[ \t]+/, "", name)
  for (v in vars) if (index(name, v "_") == 1) { print FILENAME ":" FNR ":" name; break }
}
{ doc = "" }
' $(find {SUITE_DIR} -name '*.sol' ! -path '{SUITE_DIR}/utils/MockERC20.sol') \
  pass=2 {SUITE_DIR}/Base.sol pass=3 {SUITE_DIR}/handlers/*.sol
```

Clear a hit in one of two ways:

- Stand-in: add the `// stand-in limits:` block directly above its declaration and enforce each bound that isn't `none` inside the stand-in, with a `require` or a cap.
- Handler: clamp it to the stand-in's block (`handler-patterns.md`, Stand-in Handlers). The hit clears once the stand-in carries its block.
