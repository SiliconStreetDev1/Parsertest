```
server:
  customMiddleware:
    - name: fiori-tools-servestatic
      beforeMiddleware: fiori-tools-proxy
      configuration:
        paths:
          # Notice we are NOT using /resources here!
          - path: /custom-local-lib/nz/co/siliconst/ui5/metaui
            src: ./node_modules/@siliconst/metaui/dist   # Make sure this points to the compiled JS files
            fallthrough: false


"sap.ui5": {
    "resourceRoots": {
      "nz.co.siliconst.ui5.metaui": "/custom-local-lib/nz/co/siliconst/ui5/metaui"
    }

```
