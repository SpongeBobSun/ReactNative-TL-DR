NativeModules

-----------------

in [RCTCxxBridge start]

    [self _initModules:RCTGetModuleClasses withDispatchGroup:prepareBridge lazilyDiscovered:NO];

        [self registerModuleForClasses]

          * generate module data

             RCTModuleData.mm: Create module

          * add module data to `_moduleDataByID`

       Create module instance in `instance` getter.

  in RCTBridge

    Record RCT_EXPORT_MODULE macro classes



[RCTCxxBridge _initializeBridge]

reactInstance->initializeBridge

    [RCTCxxBridge buildModuleRegistry]

    createNativeModules(_moduleDataByID, self, _reactInstance) (in file RCTCxxUtils)

