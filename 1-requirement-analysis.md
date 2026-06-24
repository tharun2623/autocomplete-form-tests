import { Page, Locator, expect } from '@playwright/test';

/**
 * Page Object Model for the Autocomplete Form.
 * URL: https://test.com/autocomplete-form
 *
 * Encapsulates all selectors and interaction methods so test scripts
 * remain readable and maintainable.
 */
export class AutocompleteFormPage {
  readonly page: Page;

  // Locators
  readonly inputField: Locator;
  readonly suggestionList: Locator;
  readonly suggestionItems: Locator;
  readonly nextButton: Locator;
  readonly errorMessage: Locator;
  readonly successContainer: Locator;

  constructor(page: Page) {
    this.page = page;
    this.inputField = page.locator('#input-field');
    this.suggestionList = page.locator('ul.suggestions');
    this.suggestionItems = page.locator('ul.suggestions li');
    this.nextButton = page.locator('#next-button');
    this.errorMessage = page.locator('span.error-message');
    this.successContainer = page.locator('div.success-container');
  }

  /**
   * Navigate directly to the autocomplete form page.
   */
  async goto() {
    await this.page.goto('/autocomplete-form');
  }

  /**
   * Type text into the input field character by character to trigger
   * the filtering event listeners.
   */
  async typeInInput(text: string) {
    await this.inputField.click();
    await this.inputField.fill(text);
  }

  /**
   * Clear the input field using keyboard Escape.
   * Assumption: Escape clears the field and resets suggestions.
   */
  async clearWithEscape() {
    await this.inputField.focus();
    await this.page.keyboard.press('Escape');
  }

  /**
   * Select a suggestion from the list by its exact visible text.
   */
  async selectSuggestion(text: string) {
    await this.suggestionList.locator(`li:has-text("${text}")`).click();
  }

  /**
   * Submit the form by clicking the Next button.
   */
  async clickNext() {
    await this.nextButton.click();
  }

  /**
   * Submit the form by pressing Enter while the Next button is focused.
   */
  async submitWithEnter() {
    await this.nextButton.focus();
    await this.page.keyboard.press('Enter');
  }

  /**
   * Returns an array of the visible suggestion texts.
   */
  async getVisibleSuggestions(): Promise<string[]> {
    const items = await this.suggestionItems.all();
    const texts: string[] = [];
    for (const item of items) {
      const isVisible = await item.isVisible();
      if (isVisible) {
        const text = await item.innerText();
        texts.push(text.trim());
      }
    }
    return texts;
  }

  /**
   * Assert that the error message is visible.
   */
  async assertErrorVisible() {
    await expect(this.errorMessage).toBeVisible();
  }

  /**
   * Assert that the success container is visible.
   */
  async assertSuccessVisible() {
    await expect(this.successContainer).toBeVisible();
  }

  /**
   * Assert that the error message is NOT visible.
   */
  async assertErrorHidden() {
    await expect(this.errorMessage).toBeHidden();
  }

  /**
   * Assert that the success container is NOT visible.
   */
  async assertSuccessHidden() {
    await expect(this.successContainer).toBeHidden();
  }

  /**
   * Assert the input field has a specific value.
   */
  async assertInputValue(expected: string) {
    await expect(this.inputField).toHaveValue(expected);
  }

  /**
   * Assert exact visible suggestions match the expected list.
   */
  async assertVisibleSuggestions(expected: string[]) {
    const visible = await this.getVisibleSuggestions();
    expect(visible).toEqual(expected);
  }
}
