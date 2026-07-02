# Parsertest

sap.ui.define([], function () {
  "use strict";

  /**
   * Formatter utility for SAPUI5.
   * Provides fault-tolerant string formatting for JSON and XML data.
   */
  return {
    /**
     * Formats a JSON string.
     * @param {string} rawJson - The input JSON string.
     * @returns {string} - Pretty-printed JSON or original input on failure.
     */
    formatJson: function (rawJson) {
      if (!rawJson || typeof rawJson !== "string") {
        return rawJson;
      }

      try {
        const parsedData = JSON.parse(rawJson);
        return JSON.stringify(parsedData, null, 2);
      } catch (error) {
        // Fallback to original string on parsing error
        return rawJson;
      }
    },

    /**
     * Formats an XML string with indentation.
     * @param {string} rawXml - The input XML string.
     * @returns {string} - Indented XML or original input on failure.
     */
    formatXml: function (rawXml) {
      if (!rawXml || typeof rawXml !== "string") {
        return rawXml;
      }

      try {
        let formatted = "";
        let indent = "";
        const tab = "  ";

        // Tokenize by tags keeping delimiters
        const tokens = rawXml.split(/(<[^>]*>)/g).filter(function (token) {
          return token.trim().length > 0;
        });

        tokens.forEach(function (token) {
          if (token.startsWith("</")) {
            // Closing tag: reduce indent
            indent = indent.slice(tab.length);
            formatted += "\n" + indent + token;
          } else if (token.startsWith("<") && !token.endsWith("/>")) {
            // Opening tag: add tag, increase indent
            formatted += "\n" + indent + token;
            indent += tab;
          } else {
            // Self-closing or text node: just add with current indent
            formatted += (formatted ? "\n" : "") + indent + token;
          }
        });

        return formatted.trim();
      } catch (error) {
        // Fallback to original string on processing error
        return rawXml;
      }
    }
  };
});
