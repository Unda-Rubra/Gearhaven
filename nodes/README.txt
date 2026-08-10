Place .lua node definition files here to add them to the Sputnik node editor.
Subdirectories are scanned recursively.

File format (header comments at top of file):
  --@name     My Custom Node
  --@category Custom
  --@input    A
  --@input    B
  --@output   Out

  local a = input("A")
  local b = input("B")
  output("Out", a + b)
