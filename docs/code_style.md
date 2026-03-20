# Code Style Guide

## Formatting Tool

This project uses clang-format to enforce consistent code style. A `.clang-format` config file is provided in the project root, based on the Google C++ style guide with some customizations.

### Install clang-format

- **Windows**:
  ```powershell
  winget install LLVM
  # or with Chocolatey
  choco install llvm
  ```

- **Linux**:
  ```bash
  sudo apt install clang-format  # Ubuntu/Debian
  sudo dnf install clang-tools-extra  # Fedora
  ```

- **macOS**:
  ```bash
  brew install clang-format
  ```

### Usage

1. **Format a single file**:
   ```bash
   clang-format -i path/to/your/file.cpp
   ```

2. **Format the entire project**:
   ```bash
   # Run from project root
   find main -iname *.h -o -iname *.cc | xargs clang-format -i
   ```

3. **Check formatting before committing** (dry-run, no file modification):
   ```bash
   clang-format --dry-run -Werror path/to/your/file.cpp
   ```

### IDE Integration

- **Visual Studio Code**:
  1. Install the C/C++ extension
  2. Set `C_Cpp.formatting` to `clang-format` in settings
  3. Optional: enable format-on-save with `editor.formatOnSave: true`

- **CLion**:
  1. Go to `Editor > Code Style > C/C++`
  2. Set `Formatter` to `clang-format`
  3. Select "Use project's .clang-format file"

### Key Style Rules

- Indentation: 4 spaces
- Line width limit: 100 characters
- Brace style: Attach (opening brace on same line as control statement)
- Pointer and reference alignment: left
- Auto-sort include headers
- Class access modifier indent: -4 spaces

### Notes

1. Format all code before committing
2. Do not manually adjust auto-formatted alignment
3. To exempt a block from formatting:
   ```cpp
   // clang-format off
   // your code here
   // clang-format on
   ```

### Common Issues

1. **Formatting fails**:
   - Check if clang-format version is too old
   - Confirm file encoding is UTF-8
   - Verify `.clang-format` file syntax is correct

2. **Output doesn't match expected format**:
   - Check that the project root `.clang-format` file is being used
   - Ensure no other `.clang-format` file in a parent directory is taking precedence

For questions or suggestions, open an issue or pull request.
