```
// FIX: Prioritize the actual last generated CDS name from the UI model to prevent stale variant captures 
// when the user types a new entity name but the old request config hasn't been purged.
const sCdsName = uiModel.getProperty("/lastGeneratedCdsName") || (activeView.byId("cmbCdsName") as Input)?.getValue().trim().toUpperCase() || "";



export default class VariantOrchestrationHandler {
    private _oView: View;
    private _variantHandler: VariantHandler;
    private _generationHandler: DiagramGenerationHandler;
    private _fnGetText: (k: string, args?: any[]) => string;
    
    // FIX: Mutex flag to prevent UI5 dropdown race conditions during variant saves
    private _isSaving: boolean = false;



public async changeVariant(e: Event): Promise<void> { 
    // FIX: Abort execution if a save operation is currently overwriting the list aggregation.
    // This stops the canvas from fetching the old variant and resetting the user's screen.
    if (this._isSaving) return;

    const variantSelect = e.getSource() as Select;
    if (variantSelect) variantSelect.setValueState("None");



public async saveVariant(): Promise<void> {
    // FIX: Lock the mutex to prevent the UI5 <Select> control from firing ghost change events
    this._isSaving = true;

    if (!this._uiModel || !this._uiModel.getProperty(UiState.LAST_GENERATED_CDS)) {
        this._isSaving = false; // Release lock on early exit
        MessageToast.show(this._fnGetText("msgEmptyTitle") || "No diagram generated to save.");
        return;
    }



} finally {
        ViewStateHelper.setAppBusy(false, this._oView);
        
        // FIX: Release the mutex lock once the save and UI aggregation bindings are fully resolved
        this._isSaving = false;
    }
}


```
