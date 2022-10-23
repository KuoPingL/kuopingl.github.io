---
layout: post
title:  "Android - EditText : FAQ"
date:   2022-08-08 16:15:07 +0800
categories: [edittext, android, intermediate]
---

### Background
This post contains some FAQ



### Keyboard Related
#### Hiding Keyboard
```Java
_endValueEditText = findViewById<AppCompatEditText?>(R.id.edittext_end_value).apply {
    setOnEditorActionListener { v, actionId, event ->
        if(actionId == EditorInfo.IME_ACTION_DONE) {
            val imm = v.context.getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager
            imm.hideSoftInputFromWindow(v.windowToken, 0)
            _endValueEditText.clearFocus()
        }
        false
    }
}
```
