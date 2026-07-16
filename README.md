```
sap.ui.define([], function () {
    "use strict";

    return {
        calculateHeight: function (sValue) {
            const iMinRows = 40;
            const iRowHeightRem = 1.2; // Adjust based on your theme's CodeEditor line height

            if (!sValue) {
                return (iMinRows * iRowHeightRem) + "rem";
            }

            // Count lines by splitting on newline characters
            const iLineCount = sValue.split(/\r\n|\r|\n/).length;
            
            // Return whichever is larger: 40 or the actual line count
            const iTargetRows = Math.max(iMinRows, iLineCount);

            return (iTargetRows * iRowHeightRem) + "rem";
        }
    };
});```
