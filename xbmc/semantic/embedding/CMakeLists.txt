set(SOURCES
    EmbeddingEngine.cpp
    GPUAccelerator.cpp
    Tokenizer.cpp
)

set(HEADERS
    EmbeddingEngine.h
    GPUAccelerator.h
    GPUTypes.h
    Tokenizer.h
)

core_add_library(semantic_embedding)

find_package(ONNXRuntime QUIET)

if(ONNXRuntime_FOUND)
  target_link_libraries(semantic_embedding PRIVATE ${APP_NAME_LC}::ONNXRuntime)
  target_compile_definitions(semantic_embedding PRIVATE HAS_ONNXRUNTIME=1)
else()
  message(WARNING "ONNX Runtime not found. Semantic embedding features will run in stub mode.")
  target_compile_definitions(semantic_embedding PRIVATE HAS_ONNXRUNTIME=0)
endif()
