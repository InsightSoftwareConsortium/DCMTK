# declare additional include directories
include_directories(${dcmtls_SOURCE_DIR}/include ${LIBXML_INCDIR})

# declare executables
foreach(PROGRAM dcmp2pgm)
  DCMTK_ADD_EXECUTABLE(${PROGRAM} ${PROGRAM}.cc)
endforeach()

# make sure executables are linked to the corresponding libraries
foreach(PROGRAM dcmp2pgm)
  DCMTK_TARGET_LINK_MODULES(${PROGRAM} dcmpstat dcmsr dcmimage dcmimgle dcmnet dcmdata oflog ofstd)
endforeach()

