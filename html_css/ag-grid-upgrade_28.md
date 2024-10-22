# 1. Update AG Grid packages in package.json
npm install ag-grid-angular@^28.2.0 ag-grid-community@^28.2.0 ag-grid-enterprise@^28.2.0

# 2. Remove old AG Grid module imports if present
# Delete or comment out: import { ModuleRegistry } from '@ag-grid-community/core';
# Delete or comment out: ModuleRegistry.registerModules([...]);

# 3. Update Angular module
# In your app.module.ts or relevant module file:
import { AgGridModule } from 'ag-grid-angular';

@NgModule({
  imports: [
    AgGridModule,
    // Other imports...
  ],
  // ...
})
export class AppModule { }

# 4. Update component imports
# In your component files, update imports:
import { CellValueChangedEvent, CellKeyPressEvent } from 'ag-grid-community';
# Remove any imports from 'ag-grid-enterprise' that are now in 'ag-grid-community'

# 5. Set the license key (if using Enterprise)
# In your main.ts or app.module.ts:
import { LicenseManager } from 'ag-grid-enterprise';
LicenseManager.setLicenseKey("YOUR_LICENSE_KEY_HERE");

# 6. Update styling (if using custom themes)
# In your styles.scss or relevant style file:
@use "ag-grid-community/styles" as ag;

@include ag.grid-styles((
  theme: alpine,
  // Other theme parameters...
));

# 7. If you were using the Legacy Sass API, update your import paths and mixins
# Old:
# @import "~ag-grid-community/src/styles/ag-grid.scss";
# @import "~ag-grid-community/src/styles/ag-theme-alpine/sass/ag-theme-alpine-mixin.scss";
# .ag-theme-alpine {
#   @include ag-theme-alpine((
#     alpine-active-color: red
#   ));
# }

# New:
@use "ag-grid-community/styles" as ag;
@include ag.grid-styles((
  theme: alpine,
  alpine-active-color: red
));

# 8. Run the application and test
ng serve

# 9. If you encounter any issues, check the console for error messages
# and refer to the AG Grid v28 changelog for any breaking changes

# 10. Optional: Run AG Grid codemod to automate some upgrades (for v31+)
npx @ag-grid-devtools/cli@latest migrate --from=27
