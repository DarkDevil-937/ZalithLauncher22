package com.movtery.zalithlauncher.feature.addon

import com.movtery.zalithlauncher.game.version.download.DownloadTask
import java.io.File

// Modrinth Version Data Models
data class ModrinthVersionInfo(
    val id: String,
    val projectId: String,
    val files: List<ModrinthVersionFile>,
    val dependencies: List<ModrinthVersionDependency> = emptyList()
)

data class ModrinthVersionDependency(
    val versionId: String?,
    val projectId: String?,
    val dependencyType: String // "required", "optional", "incompatible"
)

data class ModrinthVersionFile(
    val url: String,
    val filename: String,
    val primary: Boolean = false,
    val hashes: Map<String, String> = emptyMap()
)

class AutoDependencyResolver(
    private val fetchVersionInfo: suspend (versionId: String) -> ModrinthVersionInfo?,
    private val fetchCompatibleVersionId: suspend (projectId: String, gameVersion: String, loader: String) -> String?
) {
    /**
     * Recursively resolves a mod version along with all required dependencies
     * and returns a list of DownloadTasks.
     */
    suspend fun resolveDownloadTasks(
        versionId: String,
        gameVersion: String,
        loader: String,
        modsDir: File,
        tasks: MutableList<DownloadTask> = mutableListOf(),
        visitedVersions: MutableSet<String> = mutableSetOf()
    ): List<DownloadTask> {
        // Prevent circular dependency loops
        if (!visitedVersions.add(versionId)) return tasks

        // 1. Fetch version details from the API
        val versionInfo = fetchVersionInfo(versionId) ?: return tasks

        // 2. Find primary file and create DownloadTask
        val targetFile = versionInfo.files.firstOrNull { it.primary } ?: versionInfo.files.firstOrNull()
        targetFile?.let { file ->
            val destFile = File(modsDir, file.filename)
            val sha1Hash = file.hashes["sha1"]

            val task = DownloadTask(
                urls = listOf(file.url),
                verifyIntegrity = true,
                targetFile = destFile,
                sha1 = sha1Hash,
                isDownloadable = true
            )
            tasks.add(task)
        }

        // 3. Auto-resolve required dependencies (e.g. Fabric API)
        for (dep in versionInfo.dependencies) {
            if (dep.dependencyType == "required") {
                val depVersionId = dep.versionId ?: fetchCompatibleVersionId(
                    dep.projectId ?: continue,
                    gameVersion,
                    loader
                )

                if (depVersionId != null) {
                    resolveDownloadTasks(
                        versionId = depVersionId,
                        gameVersion = gameVersion,
                        loader = loader,
                        modsDir = modsDir,
                        tasks = tasks,
                        visitedVersions = visitedVersions
                    )
                }
            }
        }

        return tasks.distinctBy { it.targetFile.absolutePath }
    }
}
