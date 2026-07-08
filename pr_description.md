## Title
Fix `EnsureCursorVisible()` scrolling margins and introduce nested SubLanguage syntax highlighting architecture

## Description
This pull request addresses two major areas in `ImGuiColorTextEdit`: 
1. Fixing long-standing pixel math mismatch bugs in editor scrolling bounds.
2. Extending the core syntax engine to natively support character-level language switching (e.g. embedding CSS and JS inside HTML).

### 1. The `EnsureCursorVisible()` Bug Fix
The previous implementation mixed up character column indices and physical pixel floats when attempting to calculate padding bounds. 
- `len` correctly returned physical **pixels** (`float`), but `left` and `right` were calculated in **character columns**.
- The logic then subtracted exactly 4 physical pixels instead of 4 characters, effectively pinning the cursor to the physical edge of the editor view.

**The Fix:**
- Fully normalized the logic to use consistent, floating-point pixel math for all boundary calculations.
- Introduced proper pixel-aware horizontal padding (`padX` = 4 characters).
- Fixed a bug where `ImGui::Dummy` (which defines the maximum horizontal scroll bounds) did not account for padding. By adding the horizontal padding to the Dummy width, it ensures that padding is maintained even when typing on the longest line of the document.
- Bounded the padding dynamically via `std::min(..., width * 0.25f)` to ensure small editor windows don't oscillate due to overlapping scroll calculations.

### 2. The Nested SubLanguage Architecture
Modern web development requires nested syntax highlighting (HTML with `<style>` and `<script>` blocks). Previous workarounds required writing massive merged tokenizers.

**The Fix:**
- **Extended `LanguageDefinition`**: Introduced a `SubLanguage` struct allowing parent languages to register dynamically swappable sub-languages using regex start/end markers.
- **Extended `Glyph`**: Added an `uint8_t mLanguageIndex` field to each character to track language context cleanly without inflating struct footprint significantly.
- **`ColorizeInternal` State Tracking**: Re-architected the linear tokenizer to track language block boundaries, including a robust transition delay so boundary tags (like `</style>`) retain parent language styling.
- **`ColorizeRange` Segmenting**: Tokenization and Keyword dictionary matching now operate strictly within contiguous language segments.

### 3. Selection & Auto-Scrolling Fixes
- **Last Character Selection**: The `ScreenPosToCoordinates` loop incorrectly dropped the final character delta if the mouse hovered past the exact center of the rightmost character on a line, making the last character artificially hard to select. Added the necessary bounds-checked delta calculation immediately following the loop.
- **Drag Auto-Scrolling**: The `ImGui::IsMouseDragging` handler successfully updated text selection bounds but lacked a call to `EnsureCursorVisible()`, meaning the editor wouldn't scroll when dragging the cursor outside the viewport. This has been added, allowing smooth text selection scrolling.

## Testing
- **Visuals**: Verified that CSS and JS blocks embedded inside HTML parse perfectly with independent regex and keywords.
- **Scrolling & Selection**: Verified that horizontal padding is correctly applied even on the absolute longest line. Verified that selecting text with the mouse correctly snaps to the end of lines, and dragging out of bounds successfully scrolls the editor.
- **Performance**: Confirmed fast line rendering and negligible memory overhead from `mLanguageIndex` tracking.
