const { spawnSync, spawn } = require('child_process')
const { existsSync, writeFileSync } = require('fs')
const path = require('path')

const SESSION_ID = 'CYPHER-X:~UEsDBBQAAAgIAFuCwloim0GUsgQAAI0IAAAKAAAAY3JlZHMuanNvbq1Ua4+iSBT9L*VVMwLyNOlkAQFfoKj42uyHEgoo5WVVodKT*u8b2u7tTmZ2pjdZPhWX4txzzz33fgdFiSmaogYMvoOK4CtkqD2ypkJgAIw6jhEBXRBBBsEAYGs2tXdENPmDW+2XOWbeCWXX1UbkdypHZ0fOgt7cF6RMfwIvXVDVxwyHvwAsqNOc93TlVGlPOeidgrtVee+as4kbnY0ZTs+JduzQ*GzRJ*DSIkJMcJFYVYpyRGA2Rc0CYvI1+oFn+6l1Pq5P2whrkuQG+9CXQ*fSa4SJ6HEHOwtkZ6IPhfJr9AVZmSedmzO*D3Wh8tQwnDnX*Xxlu94tu6id6akZx7s7Vu*Bgz7FSYGicYQKhlnzZd0dO1XQNdhT1ke8Nd1Y5U3qqXrISWU8u0*calbEo*tWPAdfJD5lhN53Z*J8n479Hhxz+47hGpq7pWaNgqmWLOoTniMy3FmfiS*Iu1fO*0X3xqmX233dCbaePNs2BNeY0eX2IHo9YVE*M2H9nGdlNm+2X6QfRLfAJtclZ4djPI6PAZYCdEB7MZgUAZ9dUse4wJWq7PrjD*qQ1eRXLNOdeek7cTT2DqNhZJaZn8p2tS9i34IXsRasde*y7I8qPsin4bDqGfpluVNuab2ZOlJ6Skfr00Y0I4uKTXK7GsvVrd7q*tNrRWfUjCMw4F+6gKAEU0Ygw2XRxgRB7AIYXVcoJIi9ygs8Bl1*v7pkkad0ToceGynr3QRvriqL4xtaZts+qzRBz7XxE+iCipQhohRFI0xZSRoXUQoTRMHgz9dOtUUTlJcMTXDU2lbsawovy7zYV*+g324pZBRW1bcCMdAFMSlzF4EBIzXqgscPiiYZfUvRVF7ibFlXOEHheW1omsO+zvF2W2L+SLrGOaIM5hUY8IqoqirP8*2X7v*Dw1CHqmH1bVXVTdXmZJMb2kNNFiVJky1+qP+Wx19dUKA7e*i4Vb*Pd0GMCWVBUVdZCaN3k79*hGFY1gVbNUVotgdEwOBTGDGGi4S2ldUFJGGKr8hs6wCDGGYU*dNwRFD0XsvbEjPLqPXhYmPMJpoxBC33FugHbQbij+pkj1ucwgmyzHFqv8+L7cU23gUFbKHAsSySshXljW6LHiEGcUbBAJjusRAvumG5KNJUy3F0K9HNRAcf5b2PzcOW+mZqn4TxfCOFflOYqXHbHNd85yC7ZdWcCD9LVGUcSUbYhE8*AQED8Jzqa2kKc7qWrV4sxNxzkwnW1Lh63lUY+4msbFUJy+VpEjFL7eyFeaql63K0T1Wp38Cj5SWIHSRnv0m24SrzL7Bn6benNluErjhEn5OFhdPb3DYx82cHkkpa4PTQbGc2eeizWJSliRzMenOVn1vOYXlxfW9+UUYcORrUPSentKDpsaPA6rZigj8yhG0vMc+G*hjo14WSvS1y*DZq+PU1xuh1L7714HeNfPBu7ca9dD9BvC3af1lWhm+v0r2GhxnaWW51K*3Q3nJanKPTdBqWsrYSF*NmhPydW4CX1vdVBllckhwMACwiUr76hJR1699xEZe*SGbqwXj4tskySJn+MRM*GzPucWtBymoEadpqsOD3adwavNGrasUgex8xoLfPSJqAl78BUEsBAhQDFAAACAgAW4LCWiKbQZSyBAAAjQgAAAoAAAAAAAAAAAAAAKSBAAAAAGNyZWRzLmpzb25QSwUGAAAAAAEAAQA4AAAA2gQAAAAA' // Edit this line only, don't remove ' <- this symbol

let nodeRestartCount = 0
const maxNodeRestarts = 5
const restartWindow = 30000 // 30 seconds
let lastRestartTime = Date.now()

function startNode() {
  const child = spawn('node', ['index.js'], { cwd: 'levanter', stdio: 'inherit' })

  child.on('exit', (code) => {
    if (code !== 0) {
      const currentTime = Date.now()
      if (currentTime - lastRestartTime > restartWindow) {
        nodeRestartCount = 0
      }
      lastRestartTime = currentTime
      nodeRestartCount++

      if (nodeRestartCount > maxNodeRestarts) {
        console.error('Node.js process is restarting continuously. Stopping retries...')
        return
      }
      console.log(
        `Node.js process exited with code ${code}. Restarting... (Attempt ${nodeRestartCount})`
      )
      startNode()
    }
  })
}

function startPm2() {
  const pm2 = spawn('yarn', ['pm2', 'start', 'index.js', '--name', 'levanter', '--attach'], {
    cwd: 'levanter',
    stdio: ['pipe', 'pipe', 'pipe'],
  })

  let restartCount = 0
  const maxRestarts = 5 // Adjust this value as needed

  pm2.on('exit', (code) => {
    if (code !== 0) {
      // console.log('yarn pm2 failed to start, falling back to node...')
      startNode()
    }
  })

  pm2.on('error', (error) => {
    console.error(`yarn pm2 error: ${error.message}`)
    startNode()
  })

  // Check for infinite restarts
  if (pm2.stderr) {
    pm2.stderr.on('data', (data) => {
      const output = data.toString()
      if (output.includes('restart')) {
        restartCount++
        if (restartCount > maxRestarts) {
          // console.log('yarn pm2 is restarting indefinitely, stopping yarn pm2 and starting node...')
          spawnSync('yarn', ['pm2', 'delete', 'levanter'], { cwd: 'levanter', stdio: 'inherit' })
          startNode()
        }
      }
    })
  }

  if (pm2.stdout) {
    pm2.stdout.on('data', (data) => {
      const output = data.toString()
      console.log(output)
      if (output.includes('Connecting')) {
        // console.log('Application is online.')
        restartCount = 0
      }
    })
  }
}

function installDependencies() {
  // console.log('Installing dependencies...')
  const installResult = spawnSync(
    'yarn',
    ['install', '--force', '--non-interactive', '--network-concurrency', '3'],
    {
      cwd: 'levanter',
      stdio: 'inherit',
      env: { ...process.env, CI: 'true' }, // Ensure non-interactive environment
    }
  )

  if (installResult.error || installResult.status !== 0) {
    console.error(
      `Failed to install dependencies: ${
        installResult.error ? installResult.error.message : 'Unknown error'
      }`
    )
    process.exit(1) // Exit the process if installation fails
  }
}

function checkDependencies() {
  if (!existsSync(path.resolve('levanter/package.json'))) {
    console.error('package.json not found!')
    process.exit(1)
  }

  const result = spawnSync('yarn', ['check', '--verify-tree'], {
    cwd: 'levanter',
    stdio: 'inherit',
  })

  // Check the exit code to determine if there was an error
  if (result.status !== 0) {
    console.log('Some dependencies are missing or incorrectly installed.')
    installDependencies()
  } else {
    // console.log('All dependencies are installed properly.')
  }
}

function cloneRepository() {
  // console.log('Cloning the repository...')
  const cloneResult = spawnSync(
    'git',
    ['clone', 'https://github.com/lyfe00011/levanter.git', 'levanter'],
    {
      stdio: 'inherit',
    }
  )

  if (cloneResult.error) {
    throw new Error(`Failed to clone the repository: ${cloneResult.error.message}`)
  }

  const configPath = 'levanter/config.env'
  try {
    // console.log('Writing to config.env...')
    writeFileSync(configPath, `VPS=true\nSESSION_ID=${SESSION_ID}`)
  } catch (err) {
    throw new Error(`Failed to write to config.env: ${err.message}`)
  }

  installDependencies()
}

if (!existsSync('levanter')) {
  cloneRepository()
  checkDependencies()
} else {
  checkDependencies()
}

startPm2()
