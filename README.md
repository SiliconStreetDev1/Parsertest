 customMiddleware:
    # 1. ADD THIS BLOCK FIRST
    - name: fiori-tools-servestatic
      beforeMiddleware: fiori-tools-proxy
      configuration:
        paths:
          - path: /resources/nz/co/siliconst/ui5/metaui
            src: ./node_modules/@siliconst/metaui/src   # Path to the library code
            fallthrough: false
