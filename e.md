cat app/src/main/java/com/yourname/githubmanager/ui/screens/editor/FileEditorScreen.kt
package com.yourname.githubmanager.ui.screens.editor              
import androidx.compose.foundation.layout.Box                     import androidx.compose.foundation.layout.fillMaxSize
import //suchin androidx.compose.foundation.layout.padding                 import androidx.compose.foundation.layout.size
import androidx.compose.material.icons.Icons                      import androidx.compose.material.icons.filled.ArrowBack
import androidx.compose.material.icons.filled.Save                import androidx.compose.material3.*
import androidx.compose.runtime.*                                 import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier                               import androidx.compose.ui.text.font.FontFamily
import androidx.compose.ui.unit.dp                                import androidx.lifecycle.compose.collectAsStateWithLifecycle
                                                                  /**
 * A full-screen text editor for plain text files.                 *
 * This screen assumes that the file is always text‑editable (binary files are
 * filtered out earlier in the file‑tree tap handler). It provides a simple
 * editing experience without syntax highlighting or line numbers – those are
 * intentionally left for a later phase.                           */
@OptIn(ExperimentalMaterial3Api::class)                           @Composable
fun FileEditorScreen(                                                 viewModel: FileEditorViewModel,
    onBackClick: () -> Unit                                       ) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()      val snackbarHostState = remember { SnackbarHostState() }
                                                                      // Show error as a snackbar whenever an error message appears.
    LaunchedEffect(state.error) {                                         state.error?.let { error ->
            snackbarHostState.showSnackbar(error)                         }
    }                                                             
    Scaffold(                                                             snackbarHost = { SnackbarHost(snackbarHostState) },
        topBar = {                                                            TopAppBar(
                title = { Text(text = state.fileName) },                          navigationIcon = {
                    IconButton(onClick = onBackClick) {                                   Icon(
                            imageVector = Icons.Default.ArrowBack,                            contentDescription = "Back"
                        )                                                             }
                },                                                                actions = {
                    // Show progress indicator while saving; otherwise a save button
                    // that is only enabled when the file is dirty.
                    if (state.isSaving) {                                                 CircularProgressIndicator(
                            modifier = Modifier.size(24.dp),                                  color = MaterialTheme.colorScheme.onSurface                                                                                     )
                    } else {                                                              IconButton(
                            onClick = { viewModel.save() },                                   enabled = state.isDirty
                        ) {                                                                   Icon(
                                imageVector = Icons.Default.Save,                                 contentDescription = "Save"
                            )                                                             }
                    }                                                             }
            )                                                             }
    ) { paddingValues ->                                                  Box(
            modifier = Modifier                                                   .fillMaxSize()
                .padding(paddingValues)                                   ) {
            if (state.isLoading) {                                                CircularProgressIndicator(
                    modifier = Modifier.align(Alignment.Center)                   )
            } else {                                                              OutlinedTextField(
                    value = state.content,                                            onValueChange = { viewModel.onContentChange(it) },                                                                                  modifier = Modifier.fillMaxSize(),
                    textStyle = LocalTextStyle.current.copy(                              fontFamily = FontFamily.Monospace
                    )                                                                 // No syntax highlighting, no line numbers – out of scope.                                                                      )
            }                                                             }
    }                                                             }
@commenti ➜ /workspaces/codespaces-blank/NewFolder/my-github-manager-app (main) $ cat app/src/main/java/com/yourname/githubmanager/ui/screens/editor/FileEditorViewModel.kt                           // File: app/src/main/java/com/yourname/githubmanager/ui/screens/editor/FileEditorViewModel.kt                                      package com.yourname.githubmanager.ui.screens.editor
                                                                  import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope                          import com.yourname.githubmanager.data.filesystem.FileSystemException                                                               import com.yourname.githubmanager.data.filesystem.ProjectFileSystem                                                                 import com.yourname.githubmanager.domain.FileNode
import kotlinx.coroutines.flow.MutableStateFlow                   import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow                        import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch                                  
data class EditorUiState(                                             val fileName: String = "",
    val content: String = "",                                         val isLoading: Boolean = true,
    val isSaving: Boolean = false,                                    val isDirty: Boolean = false,
    val error: String? = null                                     )
                                                                  /**
 * ViewModel that manages the content loading, editing, and saving of a text file.
 *                                                                 * It delegates the actual I/O to the provided [ProjectFileSystem] implementation                                                    * (SAF or local), keeping the UI completely storage-agnostic.
 *                                                                 * TODO: The current constructor approach is simple but not typical for Android.                                                     * In a real project, the ViewModel would be provided by a Hilt module or a manual                                                   * factory that can resolve the correct [ProjectFileSystem] instance based on the                                                    * file source (SAF vs extracted zip). The caller (e.g., navigation) must supply                                                     * the right dependency.
 */                                                               class FileEditorViewModel(
    private val fileNode: FileNode,                                   private val fileSystem: ProjectFileSystem
) : ViewModel() {                                                 
    private val _uiState = MutableStateFlow(EditorUiState())          val uiState: StateFlow<EditorUiState> = _uiState.asStateFlow()
                                                                      init {
        loadContent()                                                 }
                                                                      private fun loadContent() {
        viewModelScope.launch {                                               _uiState.update { it.copy(isLoading = true, error = null) }                                                                         try {
                val text = fileSystem.readText(fileNode)                          _uiState.update {
                    it.copy(                                                              fileName = fileNode.name,
                        content = text,                                                   isLoading = false,
                        isDirty = false                                               )
                }                                                             } catch (e: FileSystemException) {
                _uiState.update {                                                     it.copy(isLoading = false, error = "Failed to load file: ${e.message}")                                                         }
            }                                                             }
    }                                                             
    fun onContentChange(newContent: String) {                             _uiState.update {
            it.copy(content = newContent, isDirty = true, error = null)
        }                                                             }
                                                                      fun save() {
        val currentContent = _uiState.value.content                       viewModelScope.launch {
            _uiState.update { it.copy(isSaving = true, error = null) }
            try {                                                                 fileSystem.writeText(fileNode, currentContent)
                _uiState.update { it.copy(isSaving = false, isDirty = false) }
            } catch (e: FileSystemException) {                                    _uiState.update { it.copy(isSaving = false, error = "Failed to save: ${e.message}") }                                           }
        }                                                             }
}