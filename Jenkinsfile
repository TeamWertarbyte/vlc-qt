parallel 'Windows': {
    node('windows') {
        def vlcVersion = "3.0.4"
        def version

        stage('Checkout') {
            cleanWs()
            checkout([
                $class: 'GitSCM',
                branches: scm.branches,
                extensions: scm.extensions + [[$class: 'SubmoduleOption', parentCredentials: true]],
                userRemoteConfigs: scm.userRemoteConfigs
            ])
            version = readFile('VERSION').split("\n")[0].trim()
        }

        stage('Build') {
            // Download vlc (the SDK is only included in the 7z archive)
            bat """
                powershell -Command "&{"^
                    "\$AllProtocols = [System.Net.SecurityProtocolType]'Tls11,Tls12';"^
                    "[System.Net.ServicePointManager]::SecurityProtocol = \$AllProtocols;"^
                    "Invoke-WebRequest https://download.videolan.org/vlc/${vlcVersion}/win64/vlc-3.0.4-win64.7z -OutFile vlc.7z;"^
                "}" || exit /b
                "%ProgramW6432%\\7-Zip\\7z" x -ovlc vlc.7z -r
            """

            // Build vlc-qt
            bat """
                :: Setup build env
                if "%QT_BIN%"=="" set QT_BIN=C:\\Qt\\5.10.1\\msvc2017_64\\bin
                if "%VCVARS_BAT%"=="" set VCVARS_BAT="C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Community\\VC\\Auxiliary\\Build\\vcvars64.bat"
                call %VCVARS_BAT%

                mkdir build
                cd build

                :: Enable (somewhat) parallel builds
                set CL=/MP 

                cmake .. -GNinja -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX="..\\release" ^
                -DLIBVLC_LIBRARY="..\\vlc\\vlc-${vlcVersion}\\sdk\\lib\\libvlc.lib" -DLIBVLCCORE_LIBRARY="..\\vlc\\vlc-${vlcVersion}\\sdk\\lib\\libvlccore.lib" ^
                -DLIBVLC_INCLUDE_DIR="..\\vlc\\vlc-${vlcVersion}\\sdk\\include" -DCMAKE_PREFIX_PATH="%QT_BIN%\\.."

                ninja
                ninja install
            """
        }

        stage('Archive artifacts') {
            def zipFile = "VLC-Qt_${version}_win64_msvc2017_vlc${vlcVersion}.zip"

            // Note: Compress-Archive creates archives that only work on Windows, but that's okay because
            // the Windows build only works on Windows, too. https://github.com/PowerShell/Microsoft.PowerShell.Archive/issues/11
            bat('powershell -Command "Compress-Archive release\\* -DestinationPath ' + zipFile + '"')
            archiveArtifacts artifacts: zipFile, fingerprint: true
        }
    }
}
