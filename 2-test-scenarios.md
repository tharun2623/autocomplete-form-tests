import { test, expect } from '@playwright/test';
import { AutocompleteFormPage } from '../pages/AutocompleteFormPage';

/**
 * UI Test Suite — Autocomplete Form
 *
 * Covers:
 *  - Tab navigation
 *  - Keyboard interaction (Enter, Escape)
 *  - Suggestion filtering (prefix match)
 *  - Suggestion filtering (match anywhere)
 *  - Suggestion selection via click
 *  - Form submission (success + error)
 */

test.describe('Autocomplete Form — UI Tests', () => {
  let formPage: AutocompleteFormPage;

  test.beforeEach(async ({ page }) => {
    formPage = new AutocompleteFormPage(page);
    await formPage.goto();
  });

  // ─────────────────────────────────────────────
  // GROUP 1: Page Load
  // ─────────────────────────────────────────────

  test('TC-LOAD-01 | All 3 suggestions are visible on page load', async () => {
    await formPage.assertVisibleSuggestions([
      'agile methodology',
      'agile methodology process',
      'agile methodology process testing',
    ]);
    await formPage.assertErrorHidden();
    await formPage.assertSuccessHidden();
  });

  // ─────────────────────────────────────────────
  // GROUP 2: Suggestion Filtering — Prefix Match (FR-02)
  // ─────────────────────────────────────────────

  test('TC-FILTER-01 | Prefix match: typing "agile" keeps all 3 suggestions', async () => {
    await formPage.typeInInput('agile');
    await formPage.assertVisibleSuggestions([
      'agile methodology',
      'agile methodology process',
      'agile methodology process testing',
    ]);
  });

  test('TC-FILTER-02 | Prefix match: typing "agile methodology p" shows only 2 suggestions', async () => {
    await formPage.typeInInput('agile methodology p');
    await formPage.assertVisibleSuggestions([
      'agile methodology process',
      'agile methodology process testing',
    ]);
  });

  test('TC-FILTER-03 | Prefix match: typing "agile methodology process t" shows only 1 suggestion', async () => {
    await formPage.typeInInput('agile methodology process t');
    await formPage.assertVisibleSuggestions([
      'agile methodology process testing',
    ]);
  });

  test('TC-FILTER-04 | Prefix match: typing non-prefix text hides all suggestions', async () => {
    await formPage.typeInInput('xyz');
    const visible = await formPage.getVisibleSuggestions();
    expect(visible).toHaveLength(0);
  });

  test('TC-FILTER-05 | Prefix match: typing mid-string word "process" hides all suggestions in default mode', async () => {
    // In DEFAULT (prefix) mode, "process" is NOT a prefix of any suggestion
    await formPage.typeInInput('process');
    const visible = await formPage.getVisibleSuggestions();
    expect(visible).toHaveLength(0);
  });

  // ─────────────────────────────────────────────
  // GROUP 3: Suggestion Selection (FR-01)
  // ─────────────────────────────────────────────

  test('TC-SELECT-01 | Clicking a suggestion populates the input field', async () => {
    await formPage.selectSuggestion('agile methodology');
    await formPage.assertInputValue('agile methodology');
  });

  test('TC-SELECT-02 | Clicking "agile methodology process" populates input correctly', async () => {
    await formPage.selectSuggestion('agile methodology process');
    await formPage.assertInputValue('agile methodology process');
  });

  test('TC-SELECT-03 | Clicking a suggestion after typing partial text populates input with full suggestion', async () => {
    await formPage.typeInInput('agile meth');
    await formPage.selectSuggestion('agile methodology process');
    await formPage.assertInputValue('agile methodology process');
  });

  // ─────────────────────────────────────────────
  // GROUP 4: Form Submission (FR-04)
  // ─────────────────────────────────────────────

  test('TC-SUBMIT-01 | Selecting a suggestion and clicking Next shows success message', async () => {
    await formPage.selectSuggestion('agile methodology');
    await formPage.clickNext();
    await formPage.assertSuccessVisible();
    await formPage.assertErrorHidden();
  });

  test('TC-SUBMIT-02 | Typing invalid input and clicking Next shows error message', async () => {
    await formPage.typeInInput('random invalid text');
    await formPage.clickNext();
    await formPage.assertErrorVisible();
    await formPage.assertSuccessHidden();
  });

  test('TC-SUBMIT-03 | Submitting empty input shows error message', async () => {
    // Input is empty — no suggestion selected
    await formPage.clickNext();
    await formPage.assertErrorVisible();
  });

  // ─────────────────────────────────────────────
  // GROUP 5: Keyboard Navigation (TC-006)
  // ─────────────────────────────────────────────

  test('TC-KEY-01 | Tab key moves focus from input to suggestion list items', async ({ page }) => {
    await formPage.inputField.focus();
    await page.keyboard.press('Tab');
    // First suggestion item should receive focus
    const firstSuggestion = formPage.suggestionItems.first();
    await expect(firstSuggestion).toBeFocused();
  });

  test('TC-KEY-02 | Tab key moves focus from last suggestion to Next button', async ({ page }) => {
    // Tab through all 3 suggestions to reach the Next button
    await formPage.inputField.focus();
    await page.keyboard.press('Tab'); // → suggestion 1
    await page.keyboard.press('Tab'); // → suggestion 2
    await page.keyboard.press('Tab'); // → suggestion 3
    await page.keyboard.press('Tab'); // → Next button
    await expect(formPage.nextButton).toBeFocused();
  });

  test('TC-KEY-03 | Enter on a focused suggestion selects it and populates input', async ({ page }) => {
    await formPage.inputField.focus();
    await page.keyboard.press('Tab'); // focus first suggestion
    await page.keyboard.press('Enter'); // select it
    await formPage.assertInputValue('agile methodology');
  });

  test('TC-KEY-04 | Enter on Next button submits the form', async ({ page }) => {
    await formPage.selectSuggestion('agile methodology');
    await formPage.nextButton.focus();
    await page.keyboard.press('Enter');
    await formPage.assertSuccessVisible();
  });

  test('TC-KEY-05 | Escape clears the input field and resets suggestions', async ({ page }) => {
    await formPage.typeInInput('agile meth');
    await page.keyboard.press('Escape');
    await formPage.assertInputValue('');
    // All suggestions should return
    await formPage.assertVisibleSuggestions([
      'agile methodology',
      'agile methodology process',
      'agile methodology process testing',
    ]);
  });

  test('TC-KEY-06 | Full keyboard-only flow: Tab → select suggestion → Tab → Enter submits', async ({ page }) => {
    // Start from input
    await formPage.inputField.focus();
    await page.keyboard.type('agile');

    // Tab to first suggestion, select it
    await page.keyboard.press('Tab');
    await page.keyboard.press('Enter');

    // Verify input populated
    await formPage.assertInputValue('agile methodology');

    // Tab to Next button and submit
    await page.keyboard.press('Tab'); // skip remaining suggestions if needed
    // Keep tabbing until Next button is focused
    let focused = await page.evaluate(() => document.activeElement?.id);
    let attempts = 0;
    while (focused !== 'next-button' && attempts < 5) {
      await page.keyboard.press('Tab');
      focused = await page.evaluate(() => document.activeElement?.id);
      attempts++;
    }
    await page.keyboard.press('Enter');
    await formPage.assertSuccessVisible();
  });
});
