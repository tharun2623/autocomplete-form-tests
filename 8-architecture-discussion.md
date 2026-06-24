import axios, { AxiosResponse } from 'axios';

/**
 * API Test Suite — Autocomplete Form
 *
 * Validates the persisted API response against the FR-05 data contract.
 *
 * Covers:
 *  - Schema validation (all required fields present)
 *  - Data type validation (boolean, string, timestamps)
 *  - IETF BCP 47 locale format
 *  - suggestion_list accuracy (only matched suggestions)
 *  - Negative test cases (missing fields, invalid data)
 */

// ─────────────────────────────────────────────
// Configuration
// ─────────────────────────────────────────────

const API_BASE = process.env.API_BASE_URL ?? 'https://test.com/api';
const FORM_ENDPOINT = `${API_BASE}/form-responses`;

// The sample response from the assignment (used for contract tests)
const SAMPLE_RESPONSE = {
  account_id: '98765',
  account_email: 'test123@gmail.com',
  start_date: '2024-03-15T10:30:00Z',       // BUG: should be local time
  end_date: '2024-03-15T10:32:00Z',          // BUG: should be local time
  locale: 'en',                               // BUG: should be en-IN
  text: 'agile methodology',
  suggestion_list: 'agile methodology, agile methodology process, agile methodology process testing',
  completed: 'true',                          // BUG: should be boolean true
};

// ─────────────────────────────────────────────
// Helpers
// ─────────────────────────────────────────────

/**
 * IETF BCP 47 regex.
 * Matches: en, en-IN, zh-Hant-TW, sr-Latn, etc.
 */
const BCP47_REGEX = /^[a-z]{2,3}(-[A-Z][a-z]{3})?(-([A-Z]{2}|\d{3}))?(-[a-zA-Z0-9]{5,8}|\d[a-zA-Z0-9]{3})*$/;

/**
 * ISO 8601 timestamp with local timezone offset (not UTC Z).
 * Matches: 2024-03-15T16:00:00+05:30
 */
const LOCAL_TIMESTAMP_REGEX = /^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}[+-]\d{2}:\d{2}$/;

/**
 * ISO 8601 — accepts both UTC (Z) and offset formats.
 * Used for loose format checks.
 */
const ISO8601_REGEX = /^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(Z|[+-]\d{2}:\d{2})$/;

// ─────────────────────────────────────────────
// Test Suite
// ─────────────────────────────────────────────

describe('FR-05 — API Response Contract Tests', () => {

  // ── Section 1: Schema Validation ─────────────

  describe('Schema Validation — all required fields must be present', () => {

    test('TC-API-001 | Response contains all 8 required fields from FR-05', () => {
      const requiredFields = [
        'account_id',
        'account_email',
        'start_date',
        'end_date',
        'locale',
        'text',
        'suggestion_list',
        'completed',
      ];
      requiredFields.forEach((field) => {
        expect(SAMPLE_RESPONSE).toHaveProperty(field);
      });
    });

    test('TC-API-002 | No required field is null or undefined', () => {
      Object.entries(SAMPLE_RESPONSE).forEach(([key, value]) => {
        expect(value).not.toBeNull();
        expect(value).not.toBeUndefined();
      });
    });

  });

  // ── Section 2: Data Type Validation ──────────

  describe('Data Type Validation', () => {

    test('TC-API-003 | account_id is a string', () => {
      expect(typeof SAMPLE_RESPONSE.account_id).toBe('string');
    });

    test('TC-API-004 | account_email is a valid email string', () => {
      const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
      expect(typeof SAMPLE_RESPONSE.account_email).toBe('string');
      expect(SAMPLE_RESPONSE.account_email).toMatch(emailRegex);
    });

    test('TC-API-005 | start_date is a valid ISO 8601 timestamp', () => {
      expect(SAMPLE_RESPONSE.start_date).toMatch(ISO8601_REGEX);
    });

    test('TC-API-006 | end_date is a valid ISO 8601 timestamp', () => {
      expect(SAMPLE_RESPONSE.end_date).toMatch(ISO8601_REGEX);
    });

    test('TC-API-007 | end_date is after start_date', () => {
      const start = new Date(SAMPLE_RESPONSE.start_date).getTime();
      const end = new Date(SAMPLE_RESPONSE.end_date).getTime();
      expect(end).toBeGreaterThan(start);
    });

    test('TC-API-008 | text is a non-empty string', () => {
      expect(typeof SAMPLE_RESPONSE.text).toBe('string');
      expect(SAMPLE_RESPONSE.text.length).toBeGreaterThan(0);
    });

    test('TC-API-009 | suggestion_list is a non-empty string', () => {
      expect(typeof SAMPLE_RESPONSE.suggestion_list).toBe('string');
      expect(SAMPLE_RESPONSE.suggestion_list.length).toBeGreaterThan(0);
    });

    /**
     * DEFECT TEST — TC-API-010
     * FR-05 requires `completed` to be a BOOLEAN.
     * The sample response returns the STRING "true".
     * This test INTENTIONALLY FAILS to document the defect.
     */
    test('TC-API-010 | [DEFECT] completed must be boolean true, not string "true"', () => {
      // Validate type first
      expect(typeof SAMPLE_RESPONSE.completed).toBe('boolean'); // FAILS: actual type is 'string'
      expect(SAMPLE_RESPONSE.completed).toBe(true);             // FAILS: actual value is '"true"'
    });

  });

  // ── Section 3: Locale Format (BCP 47) ────────

  describe('Locale — IETF BCP 47 Format Validation', () => {

    test('TC-API-011 | locale matches BCP 47 structural format', () => {
      expect(SAMPLE_RESPONSE.locale).toMatch(BCP47_REGEX);
    });

    /**
     * DEFECT TEST — TC-API-012
     * FR-05 requires locale in BCP 47 format with region tag for this user (en-IN).
     * The sample response returns "en" — missing the region subtag.
     * This test INTENTIONALLY FAILS to document the defect.
     */
    test('TC-API-012 | [DEFECT] locale should be "en-IN" for an English-language user in India', () => {
      expect(SAMPLE_RESPONSE.locale).toBe('en-IN'); // FAILS: actual value is "en"
    });

  });

  // ── Section 4: Timestamp Timezone ────────────

  describe('Timestamps — Must be in user local time, not UTC', () => {

    /**
     * DEFECT TEST — TC-API-013
     * FR-05 requires timestamps in the user's LOCAL time.
     * The sample response uses UTC ("Z" suffix).
     * This test INTENTIONALLY FAILS to document the defect.
     */
    test('TC-API-013 | [DEFECT] start_date must use local timezone offset (+05:30), not UTC', () => {
      expect(SAMPLE_RESPONSE.start_date).toMatch(LOCAL_TIMESTAMP_REGEX); // FAILS: ends with Z
    });

    test('TC-API-014 | [DEFECT] end_date must use local timezone offset (+05:30), not UTC', () => {
      expect(SAMPLE_RESPONSE.end_date).toMatch(LOCAL_TIMESTAMP_REGEX); // FAILS: ends with Z
    });

  });

  // ── Section 5: suggestion_list Accuracy ──────

  describe('suggestion_list — Must contain only matched suggestions', () => {

    test('TC-API-015 | suggestion_list items are separated by ", " (comma space)', () => {
      const items = SAMPLE_RESPONSE.suggestion_list.split(', ');
      expect(items.length).toBeGreaterThan(0);
      items.forEach((item) => {
        expect(item.trim().length).toBeGreaterThan(0);
      });
    });

    test('TC-API-016 | suggestion_list contains the text the user entered', () => {
      const items = SAMPLE_RESPONSE.suggestion_list.split(', ');
      expect(items).toContain(SAMPLE_RESPONSE.text);
    });

    test('TC-API-017 | suggestion_list only contains suggestions that match the submitted text (prefix match)', () => {
      const submittedText = SAMPLE_RESPONSE.text; // "agile methodology"
      const allKnownSuggestions = [
        'agile methodology',
        'agile methodology process',
        'agile methodology process testing',
      ];
      const items = SAMPLE_RESPONSE.suggestion_list.split(', ');

      // Each item in suggestion_list must start with the submitted text (prefix match rule)
      items.forEach((item) => {
        const matchesPrefix = item.toLowerCase().startsWith(submittedText.toLowerCase());
        expect(matchesPrefix).toBe(true);
      });
    });

    /**
     * Simulated test for a scenario where only 2 suggestions should appear.
     * In Match-Anywhere mode, typing "process" matches:
     *   ✅ "agile methodology process"
     *   ✅ "agile methodology process testing"
     *   ❌ "agile methodology" (does NOT contain "process")
     *
     * The suggestion_list must NOT include "agile methodology" in this case.
     */
    test('TC-API-018 | [Match-Anywhere] suggestion_list excludes non-matching suggestions', () => {
      // Simulated response for "process" query in match-anywhere mode
      const simulatedMatchAnywhereResponse = {
        text: 'agile methodology process',
        suggestion_list: 'agile methodology process, agile methodology process testing',
      };

      const items = simulatedMatchAnywhereResponse.suggestion_list.split(', ');
      expect(items).not.toContain('agile methodology'); // must be excluded
      expect(items).toContain('agile methodology process');
      expect(items).toContain('agile methodology process testing');
    });

  });

  // ── Section 6: Negative Tests ─────────────────

  describe('Negative Tests', () => {

    test('TC-API-NEG-001 | Response with missing account_email should fail schema validation', () => {
      const invalidResponse = { ...SAMPLE_RESPONSE };
      delete (invalidResponse as any).account_email;

      const requiredFields = ['account_id', 'account_email', 'start_date', 'end_date',
        'locale', 'text', 'suggestion_list', 'completed'];

      const missingFields = requiredFields.filter(
        (field) => !(field in invalidResponse)
      );
      expect(missingFields).toContain('account_email');
    });

    test('TC-API-NEG-002 | completed as string "false" must fail boolean type check', () => {
      const invalidResponse = { ...SAMPLE_RESPONSE, completed: 'false' };
      expect(typeof invalidResponse.completed).not.toBe('boolean'); // string, not boolean
    });

    test('TC-API-NEG-003 | locale "en_IN" (underscore) must fail BCP 47 format validation', () => {
      // BCP 47 uses hyphens, not underscores
      const invalidLocale = 'en_IN';
      expect(invalidLocale).not.toMatch(BCP47_REGEX);
    });

    test('TC-API-NEG-004 | start_date after end_date is invalid', () => {
      const badResponse = {
        ...SAMPLE_RESPONSE,
        start_date: '2024-03-15T10:32:00Z',
        end_date: '2024-03-15T10:30:00Z', // earlier than start
      };
      const start = new Date(badResponse.start_date).getTime();
      const end = new Date(badResponse.end_date).getTime();
      expect(end).not.toBeGreaterThan(start); // this should be caught as invalid
    });

    test('TC-API-NEG-005 | suggestion_list containing non-matching suggestion must fail validation', () => {
      // User typed "agile methodology process", so "agile methodology" alone should NOT appear
      const badResponse = {
        text: 'agile methodology process',
        suggestion_list: 'agile methodology, agile methodology process, agile methodology process testing',
      };

      const items = badResponse.suggestion_list.split(', ');
      const inputText = badResponse.text.toLowerCase();

      // In prefix mode, every suggestion in the list must start with the input text
      const invalidItems = items.filter(
        (item) => !item.toLowerCase().startsWith(inputText)
      );
      expect(invalidItems.length).toBeGreaterThan(0); // "agile methodology" is invalid here
    });

  });

});
