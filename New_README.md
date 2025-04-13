#!/usr/bin/env node
'use strict';

const path = require('path');
const fs = require('fs-extra');
const execa = require('execa');

// Safety improvement: Use path.join to construct paths
const packagePath = path.join(__dirname, '../package.json');
const workingDir = path.join(__dirname, '../standalone');
const packageInfo = require(packagePath);
const { version, name } = packageInfo;

// Define supported platforms
const PLATFORMS = Object.freeze({
  MACOS: 'macos',
  WINDOWS: 'win.exe',
  LINUX: 'linux'
});

// Configuration constants
const REPO_URL = 'github.com/jondot/homebrew-tap';
const EXEC_OPTIONS = { 
  shell: true,
  stdio: 'inherit' // Show subprocess output
};

// Safety improvement: Validate environment variables
function validateEnvironment() {
  if (process.env.MANUAL_HB_PUBLISH && !process.env.GITHUB_TOKEN) {
    throw new Error('GITHUB_TOKEN is required when MANUAL_HB_PUBLISH is set');
  }
}

// Improved error handling: Wrapper for execa commands
async function safeExec(command, options = {}) {
  try {
    const result = await execa.command(command, { ...EXEC_OPTIONS, ...options });
    return result;
  } catch (error) {
    console.error(`Command failed: ${command}`);
    throw error;
  }
}

// Improved file operations: Added error handling
async function prepareDirectory(dirPath) {
  try {
    await fs.remove(dirPath);
    await fs.mkdir(dirPath, { recursive: true });
  } catch (error) {
    console.error(`Failed to prepare directory: ${dirPath}`);
    throw error;
  }
}

// Generate Homebrew formula template
function generateBrewFormula(sha256, version) {
  return `# frozen_string_literal: true

VER = "${version}"
SHA = "${sha256}"

class Hygen < Formula
  desc "The scalable code generator that saves you time."
  homepage "https://hygen.io"
  url "https://github.com/jondot/hygen/releases/download/v#{VER}/hygen.macos.v#{VER}.tar.gz"
  version VER
  sha256 SHA

  def install
    bin.install "hygen"
  end
end
`;
}

// Main logic
async function packageForPlatform(platform) {
  console.log(`Packaging for ${platform}...`);
  const fileName = `${name}-${platform}`;
  const tempDir = path.join(workingDir, `tar-${fileName}`);

  await prepareDirectory(tempDir);

  if (platform === PLATFORMS.WINDOWS) {
    const exePath = path.join(tempDir, 'hygen.exe');
    await fs.move(path.join(workingDir, fileName), exePath);
    await safeExec(
      `cd ${tempDir} && zip ../hygen.${platform}.v${version}.zip hygen.exe`
    );
  } else {
    const binaryPath = path.join(tempDir, 'hygen');
    await fs.move(path.join(workingDir, fileName), binaryPath);
    await safeExec(
      `cd ${tempDir} && tar -czvf ../hygen.${platform}.v${version}.tar.gz hygen`
    );
  }

  await fs.remove(tempDir);
}

async function publishToHomebrew(version) {
  console.log('Publishing to Homebrew tap...');
  const macosArchive = path.join(workingDir, `hygen.macos.v${version}.tar.gz`);
  
  const { stdout } = await safeExec(`shasum -a 256 ${macosArchive}`);
  const sha256Match = stdout.match(/([a-f0-9]+)\s+/);
  
  if (!sha256Match || sha256Match.length < 2) {
    throw new Error('Failed to calculate SHA256 checksum');
  }

  const sha256 = sha256Match[1];
  const formulaContent = generateBrewFormula(sha256, version);
  
  const tempDir = '/tmp/brew-tap';
  const formulaPath = path.join(tempDir, 'hygen.rb');

  try {
    await fs.writeFile('/tmp/hygen.rb', formulaContent);
    
    const commands = [
      `rm -rf ${tempDir}`,
      `git clone https://${REPO_URL} ${tempDir}`,
      `mv /tmp/hygen.rb ${formulaPath}`,
      `cd ${tempDir}`,
      `git config user.email "jondotan@gmail.com"`,
      `git config user.name "Dotan Nahum"`,
      `git add .`,
      `git commit -m "hygen: auto-release v${version}"`,
      `git push https://${process.env.GITHUB_TOKEN}@${REPO_URL}`
    ];

    await safeExec(commands.join(' && '));
    console.log('Publish to Homebrew completed successfully');
  } finally {
    await fs.remove('/tmp/hygen.rb');
  }
}

async function main() {
  try {
    validateEnvironment();

    // Package all platforms in parallel (if supported)
    await Promise.all(Object.values(PLATFORMS).map(packageForPlatform));

    console.log('Packaging completed. Contents:');
    console.log((await safeExec(`ls -lh ${workingDir}`)).stdout);

    if (process.env.MANUAL_HB_PUBLISH) {
      await publishToHomebrew(version);
    }
  } catch (error) {
    console.error('Error during execution:', error.message);
    process.exit(1);
  }
}

main();
