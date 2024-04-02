# indent-rainbow

**A simple extension to make indentation more readable**

以下是配置：

```bash
// For which languages indent-rainbow should be activated (if empty it means all).
"indentRainbow.includedLanguages": [] // for example ["nim", "nims", "python"]

// For which languages indent-rainbow should be deactivated (if empty it means none).
"indentRainbow.excludedLanguages": ["plaintext"]

// The delay in ms until the editor gets updated.
"indentRainbow.updateDelay": 100 // 10 makes it super fast but may cost more resources
```